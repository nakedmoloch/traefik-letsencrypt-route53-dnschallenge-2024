# Setup traefik with SSL certificates using Route53 with Let's Encrypt DNS challenge

![Static Badge](https://img.shields.io/badge/Route53-purple)
![Static Badge](https://img.shields.io/badge/Traefik-lightblue)
![Static Badge](https://img.shields.io/badge/SSL-orange)
![Static Badge](https://img.shields.io/badge/DNS_Challenge-blue)
![Static Badge](https://img.shields.io/badge/OPNsense-orange)


![Banner](https://github.com/nakedmoloch/traefik-letsencrypt-route53-dnschallenge-2024/blob/main/images/traefik-route53.png?raw=true)

## Table of contents

1. [Preparing the 'docker-compose.yml' file](https://github.com/nakedmoloch/traefik-letsencrypt-route53-dnschallenge-2024/blob/main/README.md)
2. [Setting up a local DNS record](https://github.com/nakedmoloch/traefik-letsencrypt-route53-dnschallenge-2024/blob/main/README.md)
3. [Publishing a service to the Internet](https://github.com/nakedmoloch/traefik-letsencrypt-route53-dnschallenge-2024/blob/main/README.md)
4. [Credits and acknowledgements](https://github.com/nakedmoloch/traefik-letsencrypt-route53-dnschallenge-2024/blob/main/README.md)

## Prerequisites

- Docker and docker compose installed
- A purchased domain on Route53
- An AWS user account with API access to the specified domain

> Important: This setup works with **labels** and not **providers**, which means that, in order to generate many certificates for different services, all of them must be on the same host within traefik, f.e. 1 VM with multiple docker containers running inside.

# Preparing the 'docker-compose.yml' file

Under the *environment* section, a Route53 credentials must be specified as follows:

```yml
    environment:
      - TZ=America/Argentina
      - AWS_REGION=us-east-1
      - AWS_ACCESS_KEY_ID=XXXXXX #--put TOKEN access ID
      - AWS_SECRET_ACCESS_KEY=XXXXXX #--put TOKEN access password
```

To get the **AWS_ACCESS_KEY_ID** and the **AWS_SECRET_ACCESS_KEY**, it is highly recommended to create an AWS user account with (restricted) privileges on the desired domain ONLY. To do this, a *group policy* and a *policy* must be created then as follows:

### 1. Creating a new user on the main AWS account

Under IAM > Users > Create user:
- User name: certresolver@iam (choose whatever you want)
- Click **Next**
- Permissions options: 'Add user to group'
- Under 'User groups', select 'Create group' and give it a name, f.e. *GROUP_certresolver*
- Select 'Create policy'
- Under 'Specify permissions', on 'Policy editor', select 'JSON' and paste the following configuration file:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "route53:GetChange",
            "Resource": "arn:aws:route53:::change/*"
        },
        {
            "Effect": "Allow",
            "Action": "route53:ListHostedZonesByName",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "route53:ListResourceRecordSets"
            ],
            "Resource": [
                "arn:aws:route53:::hostedzone/XXXXXX"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "route53:ChangeResourceRecordSets"
            ],
            "Resource": [
                "arn:aws:route53:::hostedzone/XXXXXX"
            ],
            "Condition": {
                "ForAllValues:StringEquals": {
                    "route53:ChangeResourceRecordSetsNormalizedRecordNames": [
                        "_acme-challenge.domain.com"
                    ],
                    "route53:ChangeResourceRecordSetsRecordTypes": [
                        "TXT"
                    ]
                }
            }
        }
    ]
}
```
> Note: See **hostedzone/XXXXXX** twice and **_acme-challenge.domain.com** once. To get the hosted zone, head up to *Route53* > *Hosted zones* > *domain* and under 'Hosted zone details' copy the 'Hosted zone ID' and replace it on the JSON file. Replace also the *_acme...domain.com* with the domain working on.

- Once modified the JSON file, click 'Next', give it a name, f.e. *POLICY_certresolver* and click on 'Create policy'
- Return to the 'Create user group' tab and under 'Permissions policies', search for the recently created policy, select it and click on 'Create user group'
- Now, you can add the user *certresolver@iam* to the group *GROUP_certresolver*, wich now has attached to it the policy *POLICY_certresolver*, granting privileges to the specified hosted zone

<img src="https://github.com/nakedmoloch/traefik-letsencrypt-route53-dnschallenge-2024/blob/main/images/Screenshot_20240623_144210.png?raw=true" width="720"/>

- Click **Next**, confirm that everything is OK, and proceed creating the user

Head up to *IAM* > *Users* > *certresolver@iam* and under 'Access key', click on **Create access key**:
- Under 'Use case', select *Command Line Interface (CLI)*, check the confirmation box at the bottom and click **Next**
- You may add a description, if not, go ahead and create the access key

> Important: **MAKE SURE TO SAVE THE PASSWORD!!!**

Now, under *IAM* > *Users* > *certresolver@iam*, an access key 1 would be created. Copy that **TOKEN ID** and, with the **TOKEN PASSWORD** as well, replace the **docker-compose.yml** *environment* section with those credentials

### 2. Deploying traefik with SSL certificates

After customizing the **docker-compose.yml** file, go ahead and run on the same directory:

```sh
$ docker network create proxy
```

```sh
$ docker-compose up -d
```

If everything worked, the **acme.json** should now have been modified with content inside.

> Note: If there's a *"certificate: null"* string, then something went wrong and the certificate did not generate. 

Set up a local DNS record for the domain, f.e. **traefik.domain.com** and visit it on the web browser, **http://traefik.domain.com**. If a traefik login jumps, and the URL redirects you to HTTPS, then it means it has worked correctly. 

## Setting up a local DNS record

### On Linux:

Modify the **/etc/hosts** file and link the **IP of traefik** with the **domain**

```sh
$ sudo vim /etc/hosts
```

```sh
192.168.1.200   traefik.domain.com
```

```sh
$ resolvectl flush-caches
```

### On OPNsense (or Unbound DNS service)

Under *Services* > *Unbound DNS* > *General* configure the followings:
- **Enable outbound:** True
- **Register DHCP Static Mappings:** True
- **Flush DNS Cache during reload:** True

Under *Services* > *Unbound DNS* > *Overrides* > *Host overrides* click 'Add' and configure the followings:
- **Enable:** True
- **Host:** traefik
- **Domain:** domain.com
- **Type:** A
- **IP address:** 192.168.1.200 (traefik's IP)
- Click 'Save' and then 'Apply' at the bottom (you may want to reload the service)

# Publishing a service to the Internet

![Static Badge](https://img.shields.io/badge/OPNsense-orange)
![Static Badge](https://img.shields.io/badge/Port_forwarding-gray)
![Static Badge](https://img.shields.io/badge/NAT-blue)
![Static Badge](https://img.shields.io/badge/Route53-purple)


> Note: This configuration guide is oriented towards **OPNsense**, specifically for publishing a **website**, but the concept is the same for any other firewall or service setup.

## Setting up OPNsense for port forwarding

Under *Firewall* > *Settings* > *Advanced* configure the followings:
- **Reflection for port forwards:** True
- **Automatic outbound NAT for Reflection:** True

Under *Firewall* > *NAT* > *Port Forward* click the 'Add' button and set up the following rules:

<img src="https://github.com/nakedmoloch/traefik-letsencrypt-route53-dnschallenge-2024/blob/main/images/Screenshot_20240623_172930.png?raw=true" width="1280"/>

> The rest leave it with default values

- Click 'Save'

> Important: Clone the rule and change every **HTTP** port to **HTTPS**, so it can allow 443 port forwarding.

The final result under the port forward rules should look like this:

<img src="https://github.com/nakedmoloch/traefik-letsencrypt-route53-dnschallenge-2024/blob/main/images/Screenshot_20240623_173726.png?raw=true" width="1280"/>

## Setting up Route53 to link the domain with our public IP

Under *Route53* > *Hosted zones* > *domain.com*, click on **Create record** and configure the followings:
- **Record name:** none
- **Record type:** A
- **Value:** Your public IP
- Click on 'Create records'

The *domain.com* hosted zone should look something like this:

<img src="https://github.com/nakedmoloch/traefik-letsencrypt-route53-dnschallenge-2024/blob/main/images/Screenshot_20240623_175454.png?raw=true" width="1280"/>

> Note: Route53 may take a while to do the linking, but it should not take more than 24hs, or even an hour. If that is the case, check out the configurations again and make sure everything is OK. If still not, check and analyze your firewall logs. 

# Credits and acknowledgements

Jim's Garage
https://www.youtube.com/@Jims-Garage

Riogezz
https://github.com/riogezz/traefik-docker

theogravity
https://forum.opnsense.org/index.php?topic=8783.0
