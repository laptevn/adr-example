# DB migration
## Status
Accepted

## Context
DB is placed in a private subnet in AWS. It's not possible to connect to it from outside world directly.<br>
We need to find the way to run DB migration as part of CD process effectively and securely.<br>
We have a bastion host that has access to DB.

## Decision Drivers
- Access to DB should be limited

## Considered Options
### SSH tunnel via bastion host on GitHub-hosted runner
#### Pros
- Requires small implementation effort
- We don't maintain and manage a runner
- We operate within GitHub Actions workflow.<br>
No extra actions outside of GitHub Actions.

#### Cons
- Traffic to DB is opened on a runner for some time.<br>
We still need authentication to use DB, traffic will be closed soon and traffic between a runner and bastion host is encrypted.<br>
This can be a problem for security audits.
- Having a bastion host becomes mandatory.<br>
Having IaC scripts we don't need bastion host for Production environment, all changes are performed in IaC scripts. But this option changes this.
- GitHub charges us for execution of this runner
- When we build often, GitHub-hosted runner can be more expensive than self-hosted runner

### A self-hosted runner
We place this runner in application subnet to have access to DB

#### Pros
- We operate within GitHub Actions workflow
- No need in SSH tunneling
- GitHub doesn't charge us for execution of this runner
- No dependency on bastion host
- Traffic to DB doesn't reach GitHub network

#### Cons
- We have to maintain and manage a runner.<br>
Keep the runner up to date and executing.
- The runner should be available for a set of GitHub domains
- Extra complexity to stop the runner when we don't run builds.<br>
If we don't want to pay for a runner doing nothing, we can stop it. We have to stop and start the runner ourselves.
- More expensive than Github-hosted runner if we build seldom
- If we don't start a runner for 30+ days, we have to manually add it to GitHub Actions
- There should be a runner per an environment

### Migration is performed by ECS task
This and the following options will touch performing DB migration on some resource that is in application subnet and has access to DB. Migration will be triggered by GitHub Actions still.<br>

#### Pros
- No need in SSH tunneling
- No dependency on bastion host
- Traffic to DB doesn't reach GitHub network
- No additional cost for infrastructure.<br>
We can reuse EC2 instances (that run our application) to execute migraton task.
- No limit on task duration

#### Cons
- We need to build Docker image with migration logic.<br>
This can be a general image that contains cloning of the latest migration scripts and running release script. A general image will be updated seldom.
- Workarounds for synchronous migration flow.<br>
ECS task is executed asynchronously.
- Deployment logic is spread between GitHub Actions and ECS task now
- Influence on business logic.<br>
When we use infrastructure (tuned to run our business logic) to run CI/CD actions, we occupy resources by something that has minimal effect on business logic. Imaging we have high load system with never stopping applications and long and heavy DB migration process - DB migration will influence on applications.

### Migration is performed by Fargate task
#### Pros
- No need in SSH tunneling
- No dependency on bastion host
- Traffic to DB doesn't reach GitHub network
- Near 0 cost for infrastructure.<br>
80 requests (i.e. migrations) per month with duration of 1 minute each and consuming 2GB RAM cost $0.06 per month.
- No limit on task duration

#### Cons
- We need to build Docker image with migration logic
- Workarounds for synchronous migration flow
- Deployment logic is spread between GitHub Actions and ECS task now

### Migration is performed by Lambda function
#### Pros
- No need in SSH tunneling
- No dependency on bastion host
- Traffic to DB doesn't reach GitHub network
- Near 0 cost for infrastructure.<br>
80 requests (i.e. migrations) per month with duration of 1 minute each and consuming 1GB RAM cost $0.
- No influence on business logic
- Synchronous execution

#### Cons
- We need to build a package (or Docker image) with migration logic
- Deployment logic is spread between GitHub Actions and Lambda now
- Maximum duration is 15 minutes

## Decision
Migration is performed by Lambda function.<br>
Secure, lightweight and cheap approach. We don't expect DB migration taking longer than 15 minutes.