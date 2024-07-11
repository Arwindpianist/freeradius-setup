
# Setting Up FreeRADIUS with LDAP and MySQL Authentication

## 1. Install FreeRADIUS
```bash
sudo apt-get update
sudo apt-get install freeradius freeradius-mysql freeradius-ldap
```

## 2. Configure LDAP
Edit the LDAP configuration file:
```bash
sudo nano /etc/freeradius/3.0/mods-enabled/ldap
```
Make sure the `ldap` module is configured correctly with your LDAP server details:
```plaintext
ldap {
    server = 'ldap.yourdomain.com'
    identity = 'cn=admin,dc=yourdomain,dc=com'
    password = 'your_password'
    basedn = 'dc=yourdomain,dc=com'
    filter = '(uid=%{%{Stripped-User-Name}:-%{User-Name}})'
    base_filter = "(objectclass=radiusprofile)"
    # Update other configurations as needed
}
```
Enable the LDAP module:
```bash
sudo ln -s /etc/freeradius/3.0/mods-available/ldap /etc/freeradius/3.0/mods-enabled/
```

## 3. Configure MySQL
Create the MySQL database and tables as described in the requirements file:

```sql
CREATE DATABASE radius;
USE radius;

CREATE TABLE `nas` (
  `id` int(10) NOT NULL AUTO_INCREMENT,
  `nasname` varchar(128) NOT NULL,
  `shortname` varchar(32) DEFAULT NULL,
  `type` varchar(30) DEFAULT 'other',
  `ports` int(5) DEFAULT NULL,
  `secret` varchar(60) NOT NULL DEFAULT 'secret',
  `server` varchar(64) DEFAULT NULL,
  `community` varchar(50) DEFAULT NULL,
  `description` varchar(200) DEFAULT 'RADIUS Client',
  PRIMARY KEY (`id`),
  KEY `nasname` (`nasname`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;

-- Insert your NAS details here
INSERT INTO `nas` (`nasname`, `shortname`, `type`, `ports`, `secret`, `server`, `community`, `description`) VALUES
('10.0.0.5', 'huawei_client', 'huawei', 48, 'mysecret', '', 'public', 'Main gateway router'),
('10.0.0.6', 'cisco_client', 'cisco', 48, 'mysecret', '', 'public', 'Main gateway router');

CREATE TABLE `user` (
  `username` varchar(50) NOT NULL,
  `password` varchar(50) DEFAULT NULL,
  `ipv4` varchar(50) DEFAULT NULL COMMENT 'Framed-IP-Address',
  `subnet` varchar(50) DEFAULT NULL COMMENT 'Framed-IP-Netmask',
  `ipv6` varchar(50) DEFAULT NULL COMMENT 'Framed-IPv6-Pool (return this when non proxy client)',
  `speed` varchar(50) DEFAULT NULL COMMENT 'Huawei-Qos-Profile-Name Mikrotik-Rate-LimitMpd-Limit',
  `status` varchar(50) DEFAULT NULL COMMENT '0=active 1=inactive (use during auth query)',
  `sub_qos_policy_in` varchar(50) DEFAULT NULL COMMENT 'cisco-AVPair',
  `sub_qos_policy_out` varchar(50) DEFAULT NULL COMMENT 'cisco-AVPair',
  `session_time_out` varchar(50) DEFAULT NULL COMMENT 'Session-Timeout',
  `idle_time_out` varchar(50) DEFAULT NULL COMMENT 'Idle-Timeout',
  `service_type` varchar(50) DEFAULT NULL COMMENT 'Service-Type',
  `framed_protocol` varchar(50) DEFAULT NULL COMMENT 'Framed-Protocol',
  `framed_ipv6_pool` varchar(50) DEFAULT NULL COMMENT 'Framed-IPv6-Pool (return this when it is proxy client)',
  `framed_pool` varchar(50) DEFAULT NULL COMMENT 'Framed-Pool',
  `alc_delegated_ipv6_pool` varchar(50) DEFAULT NULL COMMENT 'Alc-Delegated-IPv6-Pool',
  `alc_sla_prof_str` varchar(50) DEFAULT NULL COMMENT 'Alc-SLA-Prof-Str',
  `alc_subsc_prof_str` varchar(50) DEFAULT NULL COMMENT 'Alc-Subsc-Prof-Str',
  `alc_subsc_id_str` varchar(50) DEFAULT NULL COMMENT 'Alc-Subsc-ID-Str',
  `huawei_output_average_rate` varchar(50) DEFAULT NULL COMMENT 'Output_Average_Rate',
  `huawei_input_average_rate` varchar(50) DEFAULT NULL COMMENT 'Input_Average_Rate',
  `huawei_primary_dns` varchar(50) DEFAULT NULL COMMENT 'Huawei-Primary-DNS',
  `huawei_secondary_dns` varchar(50) DEFAULT NULL COMMENT 'Huawei-Secondary-DNS',
  `huawei_ipv6_dns_server1` varchar(50) DEFAULT NULL COMMENT 'Huawei-IPV6-DNS-Server',
  `huawei_ipv6_dns_server2` varchar(50) DEFAULT NULL COMMENT 'Huawei-IPV6-DNS-Server',
  `acct_interim_interval` varchar(50) DEFAULT NULL COMMENT 'Acct-Interim-Interval',
  PRIMARY KEY (`username`)
) ENGINE=MyISAM DEFAULT CHARSET=latin1;

-- Insert your user details here
INSERT INTO `user` (`username`, `password`, `ipv4`, `subnet`, `ipv6`, `speed`, `status`, `sub_qos_policy_in`, `sub_qos_policy_out`, `session_time_out`, `idle_time_out`, `service_type`, `framed_protocol`, `framed_ipv6_pool`, `framed_pool`, `alc_delegated_ipv6_pool`, `alc_sla_prof_str`, `alc_subsc_prof_str`, `alc_subsc_id_str`, `huawei_output_average_rate`, `huawei_input_average_rate`, `huawei_primary_dns`, `huawei_secondary_dns`, `huawei_ipv6_dns_server1`, `huawei_ipv6_dns_server2`, `acct_interim_interval`) VALUES
('standard_radius_user01@myhome', 'password1', '255.255.255.254', '255.255.255.255', 'POOL-WAN1', '300M', '0', 'subscriber:sub-qos-policy-in=BNG-300M-IN', 'subscriber:sub-qos-policy-out=BNG-300M-OUT', '1209600', '1209600', 'Framed', 'PPP', 'POOL-WAN1', 'POOL_RETAIL2', 'POOL-PD1', 'DATA_C300M', 'SUB_C300', NULL, '300000000', '300000000', '115.164.14.206', '115.164.142.206', '2001:4458:4000:405::206', '2001:4458:4000:8405::206', '1200');
```
Edit the SQL module configuration:
```bash
sudo nano /etc/freeradius/3.0/mods-enabled/sql
```
Update the connection settings to match your MySQL database:
```plaintext
sql {
    driver = "rlm_sql_mysql"
    dialect = "mysql"
    server = "localhost"
    port = 3306
    login = "radius"
    password = "password"
    radius_db = "radius"
}
```
Enable the SQL module:
```bash
sudo ln -s /etc/freeradius/3.0/mods-available/sql /etc/freeradius/3.0/mods-enabled/
```

## 4. Configure the Sites-Enabled Default
Edit the `default` file to use both LDAP and SQL modules:
```bash
sudo nano /etc/freeradius/3.0/sites-enabled/default
```
In the `authorize` section, ensure both LDAP and SQL modules are included:
```plaintext
authorize {
    ldap
    if (ok) {
        update control {
            Auth-Type := ldap
        }
    }
    else {
        sql
    }
}
```
In the `authenticate` section, ensure LDAP and SQL authentication methods are included:
```plaintext
authenticate {
    Auth-Type LDAP {
        ldap
    }
    Auth-Type SQL {
        sql
    }
}
```

## 5. Restart FreeRADIUS
Restart the FreeRADIUS service to apply the changes:
```bash
sudo systemctl restart freeradius
```

## Testing
Use a RADIUS client to test the configuration:
```bash
radtest username password localhost 0 testing123
```
Replace `username`, `password`, and `testing123` with actual values from your database.

This setup allows FreeRADIUS to authenticate against LDAP first and then fall back to MySQL if the user is not found in LDAP, meeting the specified requirements.