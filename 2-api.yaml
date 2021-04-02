AWSTemplateFormatVersion: 2010-09-09
Description: Deploy a API service into an ECS cluster behind a private load balancer.
Parameters:
  ImageUrl:
    Type: String
    Default: testingimage
  Environment:
    Type: String
    Default: testevn
Conditions:
  HasNot:
    Fn::Equals: [ 'a', 'b' ]

Resources:
  NullResource:
    Type: 'Custom::NullResource'
    Condition: HasNot
Outputs:
  MyOutput:
    Value: !Ref ImageUrl
  env:
    Value: !Ref Environment