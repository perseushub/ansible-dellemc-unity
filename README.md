## EMC Unity Configuration and Management

- Requires Ansible 2.2 or newer

This Ansible module provides access to configure and manage EMC Unity storage appliance.

To use this module, follow the sample tasks in test.yml, test-1.yml, and test-2.yml. 
To run the sample tasks, change the option values to fit your system, and then run:

    ansible-playbook test.yml 
    ansible-playbook test-1.yml 
    ansible-playbook test-2.yml 

After the completion of the Ansible task in test-1.yml, your system may reboot to sync with
the NTP server. Please wait until the Unity system comes back online before running test-2.yml.

To cap the initial setup of Unity, here is the sample task in test.yml:

    - name: Initial setup
      dellemc_unity:
        unity_hostname: "192.168.0.202"
        unity_username: admin
        unity_password: Password123#
        unity_updates:
          - {resource_type: system, id: '0', isEulaAccepted: true}
        unity_password_updates:
          - {username: admin, password: Password123#, new_password: Password123!}
        unity_license_path: /home/labadmin/unity.lic

The options 'unity_hostname', 'unity_username' and 'unity_password' are mandatory for every task using the 'dellemc_unity' module. 
The task's other options does the following in the order presented below:

1. Accept the EULA of the Unity system;
2. Update the default password of the Unity system admin user;
3. Upload license to Unity.

For updating and querying the Unity system, please refer to the sample task in test-1.yml:

    - name: Updates and queries
      dellemc_unity:
        unity_hostname: "192.168.0.202"
        unity_username: admin
        unity_password: Password123!
        unity_updates:
          - {resource_type: user, name: test1, password: Welcome1!, role: administrator}
          # - {resource_type: user, id: 'user_test1', role: 'operator'}
          - {resource_type: remoteSyslog, id: '0', enabled: True, address: '192.168.0.11:515', protocol: 1, facility: 0}
          - {resource_type: dnsServer, id: '0', addresses: [10.254.66.23, 10.254.66.24]}
          - {resource_type: ntpServer, id: '0', addresses: [10.254.140.21, 10.254.140.22], rebootPrivilege: 2}
          # - {resource_type: user, id: 'user_test1', action: 'delete'}
        unity_password_updates:
          # - {username: test1, password: Welcome1!, new_password: Welcome2!}
        unity_queries:
          # - {resource_type: user, id: 'user_test1', fields: 'role.id'}
          - {resource_type: remoteSyslog, id: "0", fields: 'address,protocol,facility,enabled'}      # id parameter has to be of the string type
          - {resource_type: dnsServer, fields: "domain, addresses, origin", page: 1, per_page: 100}
          - {resource_type: ntpServer, id: "0", fields: addresses}      # id parameter has to be of the string type

For an update or query option, the 'resource_type' attribute is mandatory, and the combination of 'id' and 'action' attributes indicate what type of update or query operation is intended. For updates:

1. If 'id' is present, then it is an update of the resource instance represented by the ID. The default action is 'modify', which can be omitted. 
2. If 'id' is missing, there are two possibilities. If the 'action' attribute is present, then this is a class-level update; otherwise, if the 'action' attribute is missing, then this update is to create a new instance of the resource type.

For queries:

1. If 'id' is present, then it is a query of the specific instance; otherwise, 
2. If 'id' is missing, then it is a query of a collection of instances of the resource type.

For more details of the query/update URL parameters, see "Unisphere Management REST API Programmer's Guide". For more details of the update arguments, see "Unisphere Management REST API Reference Guide". Each parameter or argument is an attribute in an update option.

Password updates are distinct from other updates, because you can neither query nor compare passwords. Therefore, there is a separate type of update only for passwords. In this case, you only provide the username, current password, and new password of the user that you wish to update.

To delete a resource instance, you use an update option with the resource type, instance ID, and the action 'delete'. Here is the sample task in test-2.yml:

    - name: Updates and queries
      dellemc_unity:
        unity_hostname: "192.168.0.202"
        unity_username: admin
        unity_password: Password123!
        unity_updates:
          - {resource_type: user, id: 'user_test1', action: 'delete'}

To create a pool, you have to first run the action 'recommendAutoConfiguration', then create pools according to the recommendation, and later you can update the pool. The following three sample updates should be run in three tasks in sequence:

        unity_updates:
          - {resource_type: pool, action: 'recommendAutoConfiguration'}
          - {resource_type: pool, name: 'performancePool', 'addRaidGroupParameters': [{"dskGroup" : {"id" : "dg_3"}, "numDisks" : 6, "raidType" : 48879, "stripeWidth" : 0}], 'attributes': ['name']}
          - {resource_type: pool, id: pool_1, name: 'testPool', description: 'test pool'}

To configure ESRS, you have to run the following updates in a series of tasks in sequence, and then you can run a query for the ESRS parameters:

        unity_updates:
          - {resource_type: systemInformation, id: '0', contactFirstName: 'Joe', contactLastName: 'Smith', locationName: 'Test Site', contactEmail: 'joe.smith@test.com', contactPhone: '123-456-7890'}
          - {resource_type: esrsParam, id: '0', action: checkNetwork}
          - {resource_type: esrsParam, id: '0', action: requestAccessCode}
          - {resource_type: esrsParam, id: '0', action: getAvailableSites, accessCode : "243857"}
          - {resource_type: esrsParam, id: '0', "siteId" : "324546"}
        unity_queries:
          - {resource_type: esrsParam, fields: 'enabled,isCentralized,status, statusDescription::@enum(status)'}

