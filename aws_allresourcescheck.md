```
#!/bin/bash

regions=$(aws ec2 describe-regions --query "Regions[].RegionName" --output text)

echo "===== ACTIVE COST RESOURCES ====="

for region in $regions; do
  echo ""
  echo "--- REGION: $region ---"

  echo "EC2:"
  aws ec2 describe-instances --region $region \
    --query "Reservations[].Instances[].InstanceId" --output table

  echo "EBS Volumes:"
  aws ec2 describe-volumes --region $region \
    --query "Volumes[].VolumeId" --output table

  echo "Elastic IPs (Unattached will cost):"
  aws ec2 describe-addresses --region $region \
    --query "Addresses[]" --output table

  echo "NAT Gateways (VERY EXPENSIVE):"
  aws ec2 describe-nat-gateways --region $region \
    --query "NatGateways[]" --output table

  echo "Load Balancers:"
  aws elbv2 describe-load-balancers --region $region \
    --query "LoadBalancers[]" --output table

  echo "EKS Clusters:"
  aws eks list-clusters --region $region --output table

  echo "ECR Repositories (Repositories themselves are free but images can create scan/storage charges):"
  aws ecr describe-repositories --region $region --query "repositories[]" --output table

  echo "RDS Databases:"
  aws rds describe-db-instances --region $region \
    --query "DBInstances[]" --output table

  echo "S3 Buckets (storage costs):"
  aws s3api list-buckets --query "Buckets[].Name" --output table
done
```

above shell command shows all charble resources 
