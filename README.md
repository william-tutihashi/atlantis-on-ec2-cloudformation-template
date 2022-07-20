# Atlantis on EC2 Cloudformation template

Cloudformation template to create Atlantis instance AMI. This is intended do be used as a inspiration for you to adapt to your use cases, as it provides just the minimal setup to get an Atlantis instance with terraform + terragrunt running and processing GitHub webhooks. You can fork this repo and add custom workflows, Conftest Policy Checking, override cofnigs, etc.

To create the AMI, import the cloudformation template and wait until the image is ready. To do so using the AWS CLI in bash, you may adapt and run the following command:
```shell
aws cloudformation deploy \
  --template-file ./atlantis-ami-builder.yaml \
  --stack-name some-name-for-your-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    S3LogsBucketName=xxxxxx \
    AmazonLinux2AmiId=xxxxxx \
    VpcId=xxxxxx \
    SubnetId=xxxxxx \
  --tags Key1=Value1 Key2=Value2 \
  --profile your-aws-profile \
  --region the-target-aws-region
```

At time of writing (May 2022) Image Builder is pretty slow, so the image creation may take around 30 mins. You may delete cloudformation stack after deploying it, as the AMI won't be deleted in the process.


# Lauching and configuring the Atlantis server

1. Customize the AMI recipe to fit your use cases e.g. enabling SSL, custom domain name, etc
2. Follow the [Installation Guide](https://www.runatlantis.io/docs/installation-guide.html) to generate GitHub Host Access Credentials and a Webhook Secret.
3. Launch an EC2 instance using the AMI. It must have inbound access from GitHub and outbound access to internet and S3. You may use [this](https://github.com/williamtutihashi/github-webhooks-ips-security-groups-sync) strategy to keep a security group allowing inbound access from GitHub webhooks IPs up to date.
4. The next steps requires you to SSH into the instance. Run the following commands to allow your cloud infrastructure repository to send webhooks to Atlantis. replace the `github.com/company-name/cloud-infra` with your repo name.
```bash
sudo sed -i 's/REPO_PLACEHOLDER/github.com\/company-name\/cloud-infra/g' /usr/bin/run-atlantis
```
5. Run the following commands to set the GitHub user Atlantis will use to interact with GitHub. Replace XXXXXXXXXXXX with the GitHub user name.
```bash
sudo sed -i 's/USER_PLACEHOLDER/XXXXXXXXXXXX/g' /usr/bin/run-atlantis
```
6. Run the following commands to set the GitHub token for the user. Replace XXXXXXXXXXXX with the GitHub user token.
```bash
sudo sed -i 's/TOKEN_PLACEHOLDER/XXXXXXXXXXXX/g' /usr/bin/run-atlantis
```
7. Run the following commands to set the webhook secret. Replace XXXXXXXXXXXX with the webhook secret.
```bash
sudo sed -i 's/WH_SECRET_PLACEHOLDER/XXXXXXXXXXXX/g' /usr/bin/run-atlantis
```
8. Start the Atlantis service
```bash
sudo systemctl start atlantis
```
9. Enable the Atlantis service
```bash
sudo systemctl enable atlantis
```
10. Create the required webhooks in your GitHub repo by following the instructions in Atlantis [Installation Guide](https://www.runatlantis.io/docs/installation-guide.html)
11. This step depends on how you intend to provide Terraform the access to manage you AWS accounts. Take the necessary measures to do so. The following non exhaustive list shows common approaches:
 - Create and assign an instance profile to the EC2 with permission to assume the required Terraform roles from other AWS accounts. In each AWS account, create a Terraform role and allow the EC2 instance profile to assume them (This is the recommended way, as it does not requires creating IAM users in each account and sharing access credentals). In your Terraform provider block, refer to them using the `assume_role` parameter
 - Create an IAM user with programatic access to be assumed by Terraform in each of your AWS accounts and configure the AWS credentials profiles for each one in the Atlantis instance. For example, to set the AWS credentials for your production account, run `aws configure --profile production` and follow the wizard (This is not recommended, as it requires creating and sharing access credentials with cloud management privileges). In your Terraform provider block, refer to them using the `profile` parameter
12. Check if everything is working as expected by creating an PR in your repo

The Terraform provider and backend are defined in your Terraform modules in the GitHub cloud infrastructure repository and you may use Terragrunt for better abstraction. Check [this repository](https://github.com/williamtutihashi/sample-cloud-infrastructure-repo) for an example using Terragrunt, S3 as Terraform backend and multiple AWS accounts
