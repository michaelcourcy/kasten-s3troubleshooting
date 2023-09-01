# Troubleshoot connectivity issue between kasten and the s3 service

The goal of this guide is to help you troubleshoot connectivity and operation 
issue between Kasten K10 and a S3 service. You'll disciminate if the issue is 
coming from Kasten being incompatible with a non standard s3 implementation
or from a network/configuration issue. 

Many issue can be caused by reason that are not due to Kasten being incompatible with 
a non standard S3 implementation, for instance : 
- network issue, for instance firewall that drop headers between k8s and s3
- credential issue, for instance an access key or secret key was improperly 
copied
- permission issue, for instance the user does not have putObject authorization 
on the bucket
- ...

A simple way to discriminate between Kasten and the other reasons above is to 
spin up an AWS CLI client pod inside the kasten-io namespace : 

- If the client is able to do a put operation on the bucket the problem come 
from Kasten
- If not the problem is not coming from Kasten but from one of the reason above

## Lauching the test 

In order to run this test you need 
- the S3 endpoint IP or domain name
- the access key and secret key 
- the bucket name


```
kubectl -n kasten-io run --image amazon/aws-cli s3client \
  --command --  tail -f /dev/null
kubectl -n kasten-io exec -it s3client -- bash
```

Then configure your account with env variable 
```
export AWS_ACCESS_KEY_ID=AKIAQP3EERRATA4YTRX2 # Those access key and secret key are not valid
export AWS_SECRET_ACCESS_KEY=GgqnMqhtwYvYYA/UARoW35l3U1/OIRNWfPRMkdX
export AWS_DEFAULT_REGION=us-east-1
aws s3 ls
# output the list of bucket if the user has ListBucket authorization 
```

If you are not using AWS S3 service but another S3 compatible service 
you'll have to use the option `--endpoint-url`, for instance in the 
example we'll show later in this document :

```
aws s3 --endpoint-url http://minio:9000/ ls
```

If the endpoint is using https protocol but not expose valid certificate 
use the `--no-verify-ssl` option.
```
aws s3 --endpoint-url https://minio:9443/ --no-verify-ssl ls
```

## The debug option 

When the AWS CLI reports an error that you don't immediately understand, 
or produces results that you don't expect, you can get more detail about 
the error by running the command again with the --debug option. With this 
option, the AWS CLI outputs details about every step it takes to process 
your command. The details in the output can help you to determine when 
the error occurs and provides clues about where it started.

```
aws s3 ls --debug
```

## Notes about aws region and endpoint url 

This is a discussion about the aws cli s3 client when using 
aws s3 service in the different regions.

you can skip this section if you don't use aws infrastructure. 

### The default region 
There is always a default region defined for every api call 
- defined by the AWS_DEFAULT_REGION env vars or in the profile credentials 
- if not defined by the above then us-west-2 is assumed

### the region in the endpoint-url
When you use the `--endpoint-url` option in the command line region for endpoint 
are applied following this rule :  
- https://s3.amazonaws.com is for us-east-1 by default 
- https://s3.<REGION>.amazonaws.com is for <REGION>
- not defining the option `--endpoint-url` then the default region is used https://s3.<DEFAULT_REGION>.amazonaws.com


*The default region and the region used by the endpoint must match* otherwise you'll get this kind of error ! 

```
unset AWS_DEFAULT_REGION # we unset the env vars hence us-west-2 is assumed (us-west-2 is the "default default region")
aws s3 ls --endpoint-url https://s3.eu-west-3.amazonaws.com
An error occurred (AuthorizationHeaderMalformed) when calling the ListBuckets operation: The authorization header is malformed; the region 'us-west-2' is wrong; expecting 'eu-west-3'
aws s3 ls --endpoint-url https://s3.us-west-2.amazonaws.com
# success list of buckets 
export AWS_DEFAULT_REGION=eu-west-3 
aws s3 ls --endpoint-url https://s3.us-west-2.amazonaws.com
An error occurred (AuthorizationHeaderMalformed) when calling the ListBuckets operation: The authorization header is malformed; the region 'eu-west-3' is wrong; expecting 'us-west-2'
aws s3 ls --endpoint-url https://s3.eu-west-3.amazonaws.com
# success list of buckets
aws s3 ls # default region will be used in the endpoint url so they match in all cases 
# success list of bucket 
export AWS_DEFAULT_REGION=eu-west-1
aws s3 ls # default region will be used in the endpoint url so they match in all cases 
# success list of bucket 
```

Notice that all s3 api do not handles the region information the same way, some just ignore this information.

## test with a minio instance 

Thanks to [adityajoshi12](https://github.com/adityajoshi12/kubernetes-development) for sharing this simple minio deployment that I use in my repo

Deploy minio in the cluster, this is a very simple minio deployment using minio/minio123 as access and secret key.

```
kubectl create -f minio-dev.yaml
```

Once minio up and running open the console with port-forward 
```
kubectl --namespace kasten-io port-forward service/minio 9090:9090  
```

Open your browser to http://localhost:9090, connect to the interface with minio/minio123 and create the bucket test.

Now comeback to the s3client 
```
kubectl -n kasten-io exec -it s3client -- bash
export AWS_ACCESS_KEY_ID=minio
export AWS_SECRET_ACCESS_KEY=minio123
aws s3 ls --endpoint-url http://minio:9000/
```

It will output
```
2023-09-01 07:53:27 test
```

Now let's put an object in test 
```
echo "put me in test" > test.txt
aws s3 --endpoint-url http://minio:9000/ cp test.txt s3://test/folder-for-test/test.txt 
aws s3 --endpoint-url http://minio:9000/ ls s3://test/folder-for-test/
```

You should see 
```
2023-09-01 08:02:17         15 test.txt
```

## uninstall 

```
kubectl -n kasten-io delete po s3client
kubectl delete -f minio-dev.yaml
```