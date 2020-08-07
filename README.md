# AWS Setup Procedure

## EC2 
  1. create two ec2 instances for api and frontend (could be single for both)
     - 1.1. AMI: Ubuntu Server 16.04 LTS, SSD Volume Type
     - 1.2. Instance Type: t2.micro
     - 1.3. Config Detail: 
        - 1.3.1. can leave most of configs as it is (default setting)
        - 1.3.2. User Data: paste 'startup.sh' to install necessary packages (e.g., docker, docker-compose, aws-cli, incron ...)
     - 1.4. Add Storage: default setting
     - 1.5. Tags: optional
     - 1.6. Security Group: 
        - 1.6.1. SSH (as default)
        - 1.6.2. HTTP (don't need to touch 'source')
        - 1.6.3. HTTPS (don't need to touch 'source')
      - 1.7. save access pem key in your safe & local computer 
        * don't forget change the permission (sudo chmod 400 xxxx.pem)

  * you can also create template AMI for skip above setting manually

## S3 
  1. create two buckets for production & development environments
### Public Access
  1. change 'block public access'
     - 1.1. uncheck two items at bottom
     - 1.2. add a bucket policy to allow public access
      ```
      {
            "Sid": "AllowPublicRead",
            "Effect": "Allow",
            "Principal": {
                "AWS": "*"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::<bucket_name>/<target_public_folder>/*"
        },
      ```
### Cache Control
  - goals: avoid unnecessary requests or reduce the amount of requests by allowing cache at clients
#### For Existing Objects
  1. go to s3 console
  2. go to your target bucket
  3. select target files/directories
  4. choose 'More'
  5. select 'Change metadata'
  6. in the "key" field, select "Cache-Control" from the drop down menu and set 'max-age=xxx' for 'Value' (e.g., 1 year: max-age=31536000)
  7. press 'Save'
#### For New Objects
  * you might need to add 'cache-control' option for newly uploaded files programmatically using language sdk
  1. set the option when uploading files
### Lifecycle control 
  - goals: automatic delete objects with delete marker or old version, or transitions to s3 Glacier
  1. go to s3 console and select your target bucket
  2. go to 'management' tab
  3. press 'add lifecycle rule'
     - 3.1. apply your lifecycle logic (e.g., premanent deletion of objects with delete marker or old versions after a month)


## Transfer a domain to a different account
  * diagram: ================ update ===============
  - two options to do this.
    1. programmatic way
    2. AWS Support
  - this focuses on option 1
     - 1. src account setup 
        - 1.1. create api role which has full control of route53
        - 1.2. run a command:  'aws route53domains transfer-domain-to-another-aws-account --domain-name <your_target_domain> --account-id <destination_account_id> --profile <your_api_user>'
          * USE 'us-east-1' as region otherwise you get an error (i.e., see bugs#1) - you might specify '--region' in above command or change config setting for your profile
        - 1.3. you receive following output:
          ```
            {
              "OperationId": "79accf82-a752-4592-a07d-e8dd119a5e5a",
              "Password": "\"1yzDEGUx+'4)C"
            }
          ```
          * id and password values vary
      2. dest account setup 
         - 2.1. create api role which has full control of route53
         - 2.2. run a command:  'aws route53domains accept-domain-transfer-from-another-aws-account --domain-name <your_target_domain --password <given_password> --profile <your_api_user>'
            * USE 'us-east-1' as region otherwise you get an error (i.e., see bugs#1) - you might specify '--region' in above command or change config setting for your profile
            * password: use the one given from 1.3
         - 2.3. go to target domain page in route53 console of your dest account
            - 2.3.1. change nameserver values to the one in newly created hosted zone (3.1.1 so you can do this later after 3.1.1)
            - 2.3.1. (if necessary) if src account and dest account have different contact info, change it to the one of dest account
      3. migrate 'Hosted zones' for transferred domain (reference: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-migrating.html)
         - 3.1. you can achieve this by using the console or command line
            - 3.1.1. create a new public hosted zone with target domain (need to add the domain manually)
              * this creates a new NS and SOA
            - 3.1.2. move necessary record sets (e.g., 'A' record) from route53 in src account to the one in dest account EXCEPT for NS and SOA
              * choose (a) or (b) as your preference
              - 3.1.2.(a) manually migrate record sets (use if you have few record sets)
              - 3.1.2.(b) use aws-cli to migrate records sets (use if you have a huge amount of record sets)
            - 3.1.3. lower TTL for both NS record sets (could be 60sec) and wait for previous TTL to expires (usually takes 2 days)
              * this is to migrate your domain smoothly with less downtime your application
              * details: TTL: how long DNS resolver caches the information about your domain and NS. after the TTL expires, DNS resolver request to the DNS service provider for your domain to get the NS
              - this means that if you remove old record sets without the TTL exipires, your app is unanavailable until the TTL expires because DNS resolvers try to use cached information (the domain & old NS) but you remove the old NS record set.
            * in order to avoid the downtime of your app, first, lower TTL for both and wait for those TTLs to expires. after two days, you can remove old hosted zones. this time, it won't take 2 days to apply your modification since lowered TTLs
            - 3.1.4. make sure that requests go to the new hosted zones in dest account
            - 3.1.5. change the TTL for the NS record back to higher value. (e.g., 172800 sec: 2days)

## Email Service Setup


## bugs/trouble shootings
  1. aws cli: aws route53domains ...
     - error: Could not connect to the endpoint URL: "https://route53domains.<region>.amazonaws.com/"
     - problem: route53 should be global region, but there is no option for the global and no info about the region in the web interface. so I don't know which region should I choose.
     - solution: choose 'us-east-1' (I think it is default region). this should work.
  
