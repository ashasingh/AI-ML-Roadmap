.. Copyright 2010-2024 Amazon.com, Inc. or its affiliates. All Rights Reserved.

   This work is licensed under a Creative Commons Attribution-NonCommercial-ShareAlike 4.0
   International License (the "License"). You may not use this file except in compliance with the
   License. A copy of the License is located at http://creativecommons.org/licenses/by-nc-sa/4.0/.

   This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND,
   either express or implied. See the License for the specific language governing permissions and
   limitations under the License.
   
.. _aws-boto3-ec2-instance-lifecycle:

#####################################
Understanding EC2 Instance Lifecycle
#####################################

When writing automation scripts, responding to infrastructure events, or troubleshooting
behavior, it is critical to correctly handle Amazon EC2 instance state transitions. Because
AWS is a distributed system, API calls that change an instance's state (such as starting,
stopping, or terminating an instance) are asynchronous. 

This guide explains how these lifecycle transitions surface in Boto3, non-obvious behaviors 
in practice, and how to properly write reliable automation for your EC2 instances.

The Instance State Object 
=========================

Whether you use the Boto3 EC2 client or resource, you will often inspect an instance's state.
The state is typically represented as a dictionary containing a ``Code`` and a ``Name``:

* **0 (pending)**: The instance is preparing to enter the ``running`` state.
* **16 (running)**: The instance is running and ready for use.
* **32 (shutting-down)**: The instance is preparing to be terminated.
* **48 (terminated)**: The instance has been permanently deleted.
* **64 (stopping)**: The instance is preparing to be stopped.
* **80 (stopped)**: The instance is shut down and not incurring compute charges.

Common Pitfalls with Instance State
===================================

1. Eventual Consistency and Stale Data (Resource Model)

If you are using the EC2 Boto3 Resource model, it is crucial to remember that resource 
objects cache their attributes. Calling a state transition method (e.g., ``instance.stop()``) 
does *not* automatically update the ``instance.state`` attribute on your Python object.

.. code-block:: python

    import boto3
    
    ec2 = boto3.resource('ec2')
    instance = ec2.Instance('i-1234567890abcdef0')
    
    print(instance.state['Name'])  # e.g., 'running'
    instance.stop()
    print(instance.state['Name'])  # Still says 'running'!
    
    # You MUST reload the instance to get the updated status
    instance.reload()
    print(instance.state['Name'])  # Now says 'stopping'

2. Asynchronous Transitions vs. Final States

When you request an instance to terminate, the EC2 API returns an immediate response indicating 
the instance has entered the ``shutting-down`` state. It does not wait for the instance to hit 
``terminated``. Scripts the rely on an instance being fully terminated before continuing 
(like deleting a VPC) will fail if they do not wait.

Using Waiters for Reliability
=============================

The most robust way to handle asynchronous lifecycle events is by using Waiters. Boto3 Waiters
automatically poll the EC2 API and pause your script until an instance has reached a desired 
final state. 

Here are the standard EC2 instance waiters:

* ``InstanceExists``
* ``InstanceRunning``
* ``InstanceStopped``
* ``InstanceTerminated``

Practical Example: Starting an Instance Securely
------------------------------------------------

.. code-block:: python

    import boto3

    ec2_client = boto3.client('ec2')
    
    # 1. Request the instance to start
    instance_id = 'i-1234567890abcdef0'
    ec2_client.start_instances(InstanceIds=[instance_id])
    
    # 2. Get the waiter
    waiter = ec2_client.get_waiter('instance_running')
    
    # 3. Wait for the instance to be fully running
    print(f"Waiting for instance {instance_id} to be running...")
    waiter.wait(InstanceIds=[instance_id])
    print("Instance is now running!")

Practical Example: Terminating an Instance
------------------------------------------

When deleting infrastructure, relying on waiters ensures subsequent deletion API calls 
do not fail due to dependency violations:

.. code-block:: python

    import boto3

    ec2_resource = boto3.resource('ec2')
    instance = ec2_resource.Instance('i-1234567890abcdef0')
    
    # Terminate the instance
    response = instance.terminate()
    print(f"Current state: {response[0]['CurrentState']['Name']}") # 'shutting-down'
    
    # Wait until it is fully terminated
    instance.wait_until_terminated()
    
    print("Instance terminated. Safe to remove security groups or VPCs.")
    
Best Practices Summary
======================

* **Never assume immediate state changes:** API calls like ``start_instances()`` and ``stop_instances()`` only initiate the transition.
* **Always use Waiters:** When writing automation, rely on Waiters rather than writing custom loops with ``time.sleep()``.
* **Reload Resource attributes:** If you inspect object properties (``instance.state``) across time, call ``instance.reload()`` to ensure your data is fresh.
