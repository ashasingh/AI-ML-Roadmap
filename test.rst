.. Copyright 2010-2017 Amazon.com, Inc. or its affiliates. All Rights Reserved.

   This work is licensed under a Creative Commons Attribution-NonCommercial-ShareAlike 4.0
   International License (the "License"). You may not use this file except in compliance with the
   License. A copy of the License is located at http://creativecommons.org/licenses/by-nc-sa/4.0/.

   This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND,
   either express or implied. See the License for the specific language governing permissions and
   limitations under the License.

.. _aws-boto3-ec2-instance-lifecycle:

###########################
EC2 Instance Lifecycle Guide
###########################

Understanding the lifecycle of an Amazon EC2 instance is critical for building robust automation. This guide explains how instance state transitions are surfaced through Boto3, highlights non-obvious behaviors, and provides best practices for handling lifecycle changes in your scripts.

Instance States
===============

An EC2 instance transitions through several states from the moment it is launched until it is terminated.

* **pending**: The instance is being prepared. It is not yet available for login.
* **running**: The instance is fully functional and billing is active.
* **stopping**: The instance is transitioning from running to stopped.
* **stopped**: The instance is shut down but still exists. You are not billed for instance usage, but you are billed for EBS volumes.
* **shutting-down**: The instance is being prepared for termination.
* **terminated**: The instance has been permanently deleted.

Accessing State in Boto3
========================

You can check the current state of an instance using either the high-level Resource interface or the low-level Client interface.

Using Resources
---------------

The Resource interface provides a human-readable state name and a numerical code.

.. code-block:: python

    import boto3

    ec2 = boto3.resource('ec2')
    instance = ec2.Instance('i-1234567890abcdef0')

    print(f"State Name: {instance.state['Name']}")
    print(f"State Code: {instance.state['Code']}")

Using Clients
-------------

The Client interface returns the state within the ``describe_instances`` response.

.. code-block:: python

    import boto3

    client = boto3.client('ec2')
    response = client.describe_instances(InstanceIds=['i-1234567890abcdef0'])
    state = response['Reservations'][0]['Instances'][0]['State']

    print(f"State Name: {state['Name']}")

.. note::
   State codes are useful for programmatic comparisons (e.g., 16 is ``running``), while names are better for logging.

Waiters: Automating State Transitions
=====================================

A common mistake in automation is attempting to use an instance before it has fully transitioned (e.g., trying to SSH into a ``pending`` instance). Boto3 Waiters poll the instance state for you.

.. code-block:: python

    # Using Resource Waiters
    instance.start()
    instance.wait_until_running()
    print("Instance is now running!")

    # Using Client Waiters
    client.start_instances(InstanceIds=['i-1234567890abcdef0'])
    waiter = client.get_waiter('instance_running')
    waiter.wait(InstanceIds=['i-1234567890abcdef0'])

Waiters are the preferred alternative to manual ``time.sleep()`` loops because they are optimized for the specific service behavior.

Non-Obvious Behaviors and Pitfalls
==================================

When writing automation, keep these often-overlooked behaviors in mind:

IP Address Volatility
---------------------
When an instance is stopped, it releases its public IPv4 address. When it is restarted, it receives a new public IPv4 address.
* **Automation Impact**: If your script stores the IP address at launch, it will be incorrect after a stop/start cycle. Use Elastic IPs if you need a persistent address.
* **Boto3 Detail**: The ``PublicIpAddress`` attribute may be missing entirely from the ``describe_instances`` response if the instance is in a ``stopped`` state.

Attribute Latency
-----------------
Properties like ``PublicIpAddress`` or ``PublicDnsName`` are often not assigned until the instance reaches the ``running`` state.
* **Gotcha**: If you call ``instance.reload()`` immediately after ``run_instances()``, these attributes might still be empty. Always wait for the ``running`` state before fetching network metadata.

Instance Store vs. EBS
----------------------
* **Stopped State**: Only EBS-backed instances can be stopped. Instance store-backed instances can only be terminated or rebooted.
* **Data Loss**: Data on instance store volumes is lost when an instance is stopped (for EBS-backed instances with instance store volumes) or terminated.

Termination Protection
----------------------
If termination protection is enabled, Boto3 calls to ``terminate_instances()`` will fail with a ``ClientError``. Your automation should handle this or explicitly disable protection first.

Practical Example: Robust Instance Start
========================================

This example demonstrates how to safely start an instance and retrieve its public IP once it's available.

.. code-block:: python

    import boto3
    from botocore.exceptions import WaiterError

    def start_and_get_ip(instance_id):
        ec2 = boto3.resource('ec2')
        instance = ec2.Instance(instance_id)

        print(f"Starting instance {instance_id}...")
        instance.start()

        try:
            # Wait up to 10 minutes (default) for the instance to be running
            instance.wait_until_running()
            
            # Reload attributes to ensure we have the latest metadata
            instance.reload()
            
            print(f"Instance is running at: {instance.public_ip_address}")
            return instance.public_ip_address
        except WaiterError as e:
            print(f"Instance failed to start: {e}")
            return None

Common Troubleshooting
======================

* **Instance stuck in "stopping"**: This can happen if the OS is taking a long time to shut down. Waiters will eventually timeout. Your script should catch ``WaiterError`` and decide whether to force-stop or alert.
* **State Mismatch**: If you query an instance state immediately after an action, you might get the *old* state due to eventual consistency. Use waiters to ensure you are reasoning about the actual current state.
