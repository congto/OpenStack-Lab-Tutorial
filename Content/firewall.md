# Configure Firewall FWaaS Service
The Firewall as a Service (**FWaaS**) agent adds perimeter firewall management to OpenStack Networking. FWaaS uses iptables to apply firewall policy to all virtual routers within a project, and supports one firewall policy and logical firewall instance per project. FWaaS operates at the perimeter by filtering traffic at the Neutron router. This is different from Security Groups, which operate at the instance level.

On the Network node, enable the FWaaS agent by configuring the ``/etc/neutron/fwaas_driver.ini`` initialization file
```
[service_providers]
service_provider = FIREWALL:Iptables:neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver:default

[fwaas]
driver = neutron.services.firewall.drivers.linux.iptables_fwaas.IptablesFwaasDriver
enabled = True
```

Retart and the agent service on the Network node
```
# systemctl restart neutron-l3-agent
```

On the Controller node, add the service plug-in in the ``/etc/neutron/neutron.conf`` configuration file
```
[DEFAULT]
...
service_plugins = firewall
```

and restart the Neutron server
```
# systemctl restart neutron-server
```

As admin user, check the agents list
```
# source keystonerc_admin
# neutron agent-list
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
| id                                   | agent_type         | host      | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
| 31aeab90-017f-47c9-9160-a58dd3eae854 | L3 agent           | network   | :-)   | True           | neutron-l3-agent          |
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
```

As tenant user, create a couple of firewall rules to permit HTTPS (443) and deny HTTP (80) incoming traffic
```
# neutron firewall-rule-create --protocol tcp \
--destination-port 80 \
--action deny \
--name deny-http

# neutron firewall-rule-create --protocol tcp \
--destination-port 443 \
--action allow \
--name allow-https

# neutron firewall-rule-list
+--------------------------------------+-------------+--------------------------------------+----------------------+---------+
| id                                   | name        | firewall_policy_id                   | summary              | enabled |
+--------------------------------------+-------------+--------------------------------------+----------------------+---------+
| 2d9723be-6399-4735-bba6-4815b9070e56 | deny-http   |                                      | TCP,                 | True    |
|                                      |             |                                      |  source: none(none), |         |
|                                      |             |                                      |  dest: none(80),     |         |
|                                      |             |                                      |  deny                |         |
| a3a97c13-b790-413c-9957-c8933f55a49c | allow-https |                                      | TCP,                 | True    |
|                                      |             |                                      |  source: none(none), |         |
|                                      |             |                                      |  dest: none(443),    |         |
|                                      |             |                                      |  allow               |         |
+--------------------------------------+-------------+--------------------------------------+----------------------+---------+
```

Permitted protocol types are: *udp*, *tcp*, *icmp* and *any*. If the rule is protocol agnostic, use the latter value. Rules actions are: *allow*, *deny* and *reject*.

Create a firewall policy
```
# neutron firewall-policy-create \
--firewall-rules "allow-https deny-http" \
my-policy
Created a new firewall_policy:
+----------------+--------------------------------------+
| Field          | Value                                |
+----------------+--------------------------------------+
| audited        | False                                |
| description    |                                      |
| firewall_rules | a3a97c13-b790-413c-9957-c8933f55a49c |
|                | 2d9723be-6399-4735-bba6-4815b9070e56 |
| id             | 436da4c2-272b-4845-b67d-59809457dd45 |
| name           | my-policy                            |
| shared         | False                                |
| tenant_id      | 22bdc5a0210e4a96add0cea90a6137ed     |
+----------------+--------------------------------------+
```

FWaaS always adds a default deny-all rule at the lowest precedence of each policy. Consequently, a firewall policy with no rules blocks all traffic by default.

Finally, create a firewall based on the policy just created and assign the Firewall to a tenant router
```
# neutron firewall-create \
--name my-firewall \
--router my-gateway \
my-policy

# neutron firewall-show my-firewall
+--------------------+--------------------------------------+
| Field              | Value                                |
+--------------------+--------------------------------------+
| admin_state_up     | True                                 |
| description        |                                      |
| firewall_policy_id | 436da4c2-272b-4845-b67d-59809457dd45 |
| id                 | 87572a33-56f7-4665-b296-50baf9af3bbb |
| name               | my-firewall                          |
| router_ids         | 9a45bdb6-b7ff-4329-8333-7339050ebcf9 |
| status             | ACTIVE                               |
| tenant_id          | 22bdc5a0210e4a96add0cea90a6137ed     |
+--------------------+--------------------------------------+
```
