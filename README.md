# Autoscale-group

echo "creating launch configuration"

aws autoscaling create-launch-configuration \
    --launch-configuration-name $launch_Configuration_name \
    --image-id $image_id \
    --instance-type $instance_type \
    --security-groups $security_group_id \
    --key-name sundarnew --region $region

target_group_output=$(aws elbv2 create-target-group \
    --name $target_group_name \
    --protocol HTTP \
    --port 80 \
    --target-type instance \
    --vpc-id $vpd_id \
    --region $region) 

echo "Target Group Created"

echo $target_group_output

target_group_arn=$(echo $target_group_output | jq -r '.TargetGroups[0].TargetGroupArn')

echo "Creating auto scaling group"

echo $target_group_arn

vpc_zone_identifier=$(echo $subnets_list | sed -e "s/ /,/g")
echo $vpc_zone_identifier

echo $region

aws autoscaling create-auto-scaling-group \
    --auto-scaling-group-name $auto_scaling_group_name \
    --min-size 1 \
    --max-size 5 \
    --launch-configuration-name $launch_Configuration_name \
    --target-group-arns $target_group_arn \
    --health-check-type ELB \
    --health-check-grace-period 600 \
    --vpc-zone-identifier "$vpc_zone_identifier" \
    --region "us-east-1"
    


echo "Auto Scaling group created"

echo "creating load balancer"

load_balancer_output=$(aws elbv2 create-load-balancer \
    --name $load_balancer_name \
    --subnets subnet-0e6b39e24721e5368 subnet-077ac95f4eb2bfab2 \
    --security-groups $security_group_id \
    --region $region)

echo "load balancer created"

load_balancer_arn=$(echo $load_balancer_output | jq -r '.LoadBalancers[0].LoadBalancerArn')

echo "attaching target group to load balancer"

aws autoscaling attach-load-balancer-target-groups \
    --auto-scaling-group-name $auto_scaling_group_name \
    --target-group-arns $target_group_arn \
    --region $region

echo "Done target group load balancer attachment"

echo "add litener to load balancer"

aws elbv2 create-listener --load-balancer-arn $load_balancer_arn \
--protocol HTTP --port 80  \
--default-actions Type=forward,TargetGroupArn=$target_group_arn \
--region $region
