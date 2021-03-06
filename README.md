## TODO
Upgrade Ubuntu 14.04 image to be 18.04. 
- jmeter-server service on both slaves and boss need to be upgraded from Upstart to Systemd. See this for details: https://wiki.ubuntu.com/SystemdForUpstartUsers
Fix ssh credentials to be a sustainable solution without a PW in repo (generate keys? does it work for slaves?)
Cleanup unneccesary parameters in run script
run.properties adapted
Describe possible strategies for running, who is involved and what to fix?
Add fixes so that they are possible to re-use?


## Background
These scripts create an infrastructure for running a distributed performance test with jmeter using Azure resources. It has been run with a total of 656 CPU cores on 40 D5 instances.

## Prerequisites

### MSDN subscription
Apart from permissions, also enough credits on a credit card. =)

#### Correct resource providers permissions
Subscription > Resource providers needed:
Microsoft.Network
Microsoft.Storage
Microsoft.Compute

#### Azure Cli installed
https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest

#### Resource limits
In a previous project we had scenarios that needed 600 CPU cores. Make sure to ask Microsooft to raise this limit with enough timeframe. Test the limits with the validate command below. 

Less than 200 CPU cores is handled more promptly, if you need more there is a more thorough audit process needed. 
Ask for more cores in the Dv2-series. 

### In portal, create virtual network with subnet (TODO: Script this)
### If you're using PowerShell you can do this in cli aswell, see below
virtual network: es-vnet: 10.0.0.0/16
subnet: jmeter: 10.0.4.0/24

or via PS:
New-AzureRmResourceGroup -Name jmeter -Location NorthEurope 
$virtualNetwork = New-AzureRmVirtualNetwork -ResourceGroupName jmeter -Location NorthEurope -Name es-vnet -AddressPrefix 10.0.0.0/16
$subnetConfig = Add-AzureRmVirtualNetworkSubnetConfig -Name jmeter -AddressPrefix 10.0.4.0/24 -VirtualNetwork $virtualNetwork
$virtualNetwork | Set-AzureRmVirtualNetwork

get the boss ip:
Get-AzureRmPublicIpAddress -Name jmeter-pip -ResourceGroupName jmeter

## Setup infrastructure in bash
Validate template:
`time az group deployment validate --resource-group jmeter --template-file elasticsearch-jmeter/azuredeploy.json --parameters @elasticsearch-jmeter/azuredeploy.parameters.json`

Deploy from root: 
`time az group deployment create --resource-group jmeter --template-file elasticsearch-jmeter/azuredeploy.json --parameters @elasticsearch-jmeter/azuredeploy.parameters.json`

## If you are a avid windows user and like to work with powershell do this:
Connect your account:<br />
`Login-AzureRmAccount`

Select the subscription:<br />
`Select-AzureRmSubscription -SubscriptionName {subName}`

Create ResoruceGroup in azure portal<br />
`New-AzureRmResourceGroup -Name jmeter -Location NorthEurope`<br />
`$virtualNetwork = New-AzureRmVirtualNetwork -ResourceGroupName jmeter -Location NorthEurope -Name es-vnet -AddressPrefix 10.0.0.0/16`<br />
`$subnetConfig = Add-AzureRmVirtualNetworkSubnetConfig -Name jmeter -AddressPrefix 10.0.4.0/24 -VirtualNetwork $virtualNetwork`<br />
`$virtualNetwork | Set-AzureRmVirtualNetwork`

Test the template:<br />
`Test-AzureRmResourceGroupDeployment -ResourceGroupName jmeter -TemplateFile .\azuredeploy.json -TemplateParameterFile .\azuredeploy.parameters.json` (Error if something is wrong otherwise no output)

Deploy the machines based upon the template: (In here you set the machine spec and how many "slave" nodes that will be running the jmx template)<br />
`New-AzureRmResourceGroupDeployment -Name azuredeploy -ResourceGroupName jmeter -TemplateFile .\azuredeploy.json -TemplateParameterFile .\azuredeploy.parameters.json`

Delete deployment via PS<br />
`Remove-AzureRmResourceGroup -Name jmeter`

### Delete deployment via script
time az group deployment delete --resource-group jmeter --name azuredeploy

Remove-AzureRmResourceGroup -Name jmeter

### Troubleshooting infrastructure
Installation log:
`sudo -i`
`cat /var/lib/waagent/custom-script/download/stdout`

`cat /var/lib/waagent/custom-script/download/0/stdout`

Remote hosts list is located in /opt/jmeter/apache-jmeter-3.1/bin/jmeter.properties
remote_hosts=10.0.4.20,10.0.4.21,10.0.4.22,10.0.4.23,10.0.4.24,10.0.4.25,10.0.4.26,10.0.4.27,10.0.4.28,10.0.4.29

## Run test:
ssh to boss-node (via putty if you are on windows)

Go to /opt/jmeter
run sudo ./run.sh


$ time az group deployment create --resource-group jmeter --template-file elasticsearch-jmeter/azuredeploy.json --parameters @elasticsearch-jmeter/azuredeploy.parameters.json
Long running operation wait cancelled.  Correlation ID: 34c23690-abfe-4d32-b284-4a8481e3f092

real	47m29.658s
user	0m3.106s
sys	0m0.315s

$ time az group deployment create --resource-group jmeter --template-file elasticsearch-jmeter/azuredeploy.json --parameters @elasticsearch-jmeter/azuredeploy.parameters.json
Long running operation wait cancelled.  Correlation ID: 5bed07fa-9586-4950-91db-c0385954a77b

real	48m52.389s
user	0m3.274s
sys	0m0.449s

NO VMs Created!
$ time az group deployment create --resource-group jmeter --template-file elasticsearch-jmeter/azuredeploy.json --parameters @elasticsearch-jmeter/azuredeploy.parameters.json
Long running operation wait cancelled.  Correlation ID: 2f93fa58-2fae-4796-a6fc-707958aa62fe

real	53m40.419s
user	0m3.562s
sys	0m0.365s

## Runs
https://rpm.newrelic.com/accounts/1649385/applications/54984775?tw%5Bend%5D=1508939904&tw%5Bstart%5D=1508938032#tab_hosts=breakout

40 D5 instances
${__P(Constant_Timer_ms, 30000)}
${__P(PhantomJS_Browsing_Users,1000)}
${__P(PhantomJS_Ramp_Up_Period_sec,1200)}
${__P(PhantomJS_Thread_Loop_Count, 2)}

## Windows quirks
To be able to add files to the boss node download putty and e.g. WinSCP

- Upload the wanted files (run.sh and JMeter_PhantomJS_Template.jmx from local testpack folder) via WinSCP to the home/lyko folder on the boss node
- Via putty go to the folder opt/jmeter and run the following to copy files from home/lyko : sudo cp /home/lyko/* .
- Make the run.sh executable by sudo chmod +x run.sh 
- If needed also run sudo dos2unix run.sh to fix windows to unix line endings (install via sudo apt-get install dos2unix)
- Run the wanted amount of users (configured in JMeter_PhantomJS_Template.jmx) on the configured machines via sudo ./run.sh
- Monitor the machines

## Problems installing Jmeter
Sometimes jmeter is not installed on the subnodes which makes the deployment fail. You cannot run the load test either then as the subnodes are not all capable of running the jmeter template. A workaround for this is to find which subnodes that are not installed correctly and remove their ip-adresses from the remote_hosts list in the file /opt/jmeter/apache-jmeter-3.1/bin/jmeter.properties
