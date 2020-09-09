---
layout: default
---


## AWS ELB IP dynamically resolve issue on Nginx

#### The problem

We were using Nginx (free version) as a reverse proxy in front of AWS Elastic Load Balancer to proxy all requests to the servers behind our load balancer. Now with ELB, you are supposed to use the CNAME record of it since the IPs of ELBs can and do keep changing. Usually what happens on AWS side is if your ELB starts to get traffic increase, AWS changes the machine to add a heavy beefy machine behind the scene which can cause IPs to change and thier can be other reasons also like maintenance etc.
Anyways Nginx resolves the address defined in the proxy_pass directive just once when it starts up or reloaded. For the rest of the times it keeps using the local cache. This caused the problem for us as after some time or days the IP changed and we started getting `Error: upstream timed out`.


#### The solution

Now a very easy solution is to get the NGINX plus version which is ofcourse a paid one and comes with the abililty to dynamically resolve the IPs.

But we like to be extremely cost effective found a way to make it work with the free vanilla nginx as well.

We added a resolver directive at the start of nginx conf, pointing to the AWS or self-managed DNS server in your VPC. This makes it to honour the default ttl of load balancer DNS.

Next, to make it resolve the IPs everytime, we changed the proxy_pass directive to use a variable for the DNS value. Nginx evaluates the value of the variable in each request. By setting the address as a variable and using the variable in the proxy_pass directive, we force Nginx to resolve the correct load balancer address on every request instead of at the startup or reload time.


```bash
location / {
    resolver 10.10.0.2;
    set $my_server internal-balancer-loader-xxxx.eu-west-1.elb.amazonaws.com;
    proxy_pass $my_server;
}
```


Cheers! 