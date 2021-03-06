# Task Retirement<a name="task-retirement"></a>

A task is scheduled to be retired when AWS detects the irreparable failure of the underlying hardware hosting the task\. When a task reaches its scheduled retirement date, it is stopped or terminated by AWS\.

If the task is part of a service, then the task is automatically stopped and the service schedule starts a new one to replace it\. If you are using standalone tasks, then you receive notification of the task retirement\.

## Identifying Tasks Scheduled for Retirement<a name="task-retirement-identify"></a>

If your task is scheduled for retirement, you receive an email before the event with the task ID and retirement date\. This email is sent to the address that's associated with your account, the same email address that you use to log in to the AWS Management Console\. If you use an email account that you do not check regularly, then you can use the [AWS Personal Health Dashboard](https://aws.amazon.com/premiumsupport/phd/) to determine if any of your tasks are scheduled for retirement\. To update the contact information for your account, go to the [Account Settings](https://console.aws.amazon.com/billing/home?#/account) page\.

## Working with Tasks Scheduled for Retirement<a name="task-retirement-working"></a>

If the task is part of a service, then the task is automatically stopped and the service schedule starts a new one to replace it\. If you are using standalone tasks, then you can start a new task to replace it\. For more information, see [Running Tasks](ecs_run_task.md)\.