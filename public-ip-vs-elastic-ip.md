Public IP is an IP that is attached with the EC2 instance automatically from the Amazon pool of IP address.

When you stop the instance here, the IP would again get back to the pool of IP and when you restart the instance again a random IP will be allotted to the instance.

Here the IP is associated with the instance. Once the instance is stopped or terminated the IP is gone.

Whereas the Elastic IP is a static IP that is created manually. It does not change while restarting the instance.

It is known as elastic as it can be detached from the instance and attached to another. The major advantage of this is that we can mask the failure of the instance rapidly by detaching from the instance and giving it to the other instance.

It is charged in hourly bases even when it is not associated with the running instance as it is linked with the account of the user and not on the instance.

Elastic IPs are region specific. By default, all the accounts in AWS are limited to only 5 Elastic IPs.
