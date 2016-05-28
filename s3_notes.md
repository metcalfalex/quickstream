## Save data to S3 as CSV

### Create S3 Bucket

Amazon AWS - S3

Create bucket:  
e.g.  
20160528nifiout

Bucket Properties > Permissions > Edit Bucket Policy  
Use the following permission to make all items in your bucket public.  
Replace 20160528nifiout with the name of your bucket.  

```code
{
	"Statement": [
		{
			"Sid": "AllowPublicRead",
			"Effect": "Allow",
			"Principal": {
				"AWS": "*"
			},
			"Action": "s3:GetObject",
			"Resource": "arn:aws:s3:::20160528nifiout/*"
		}
	]
}
```

Upload data_for_import_01.csv to bucket

View properties:  
Link e.g  
https://s3-us-west-2.amazonaws.com/20160528nifiout/data_for_import_01.csv

Test out link in web browser to make sure upload successful and permissions are public


### NIFI PutS3Object

Bucket: 20160528nifiout  
Region: us-west-2 (default)  

...