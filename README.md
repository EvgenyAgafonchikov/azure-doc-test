# Limitations of code generation & how to overcome them

Unfortunately, generator can not generate code for processing complex properties such as an array of objects. In that case developer has to extend auto generated code by adding handwritten code for processing complex properties. Following text will explain how to write hybrid solution using Web Application Firewall Config as an example.

*Please make sure you've read [documentation](https://github.com/Azure/azure-xplat-cli/blob/dev/Documentation/) (especially [the part about writing cmdlets](https://github.com/Azure/azure-xplat-cli/blob/dev/Documentation/Writing-Cmd.md)) for `azure-xplat-cli`*

## Examining auto generated code

WAF Config's JS code is generated into `ApplicationGateways-webApplicationFirewallConfiguration._js` file. As you look throught it to check if everything generated correctly, you'll see that `create` and `set` commands have an option `disabled-rule-groups`.

```javascript
.option('--disabled-rule-groups [disabled-rule-groups]', $('the disabled rule groups'))
// ...
if (options.disabledRuleGroups) {
  parameters.webApplicationFirewallConfiguration.disabledRuleGroups = options.disabledRuleGroups;
}
```

According to [API specifications](https://github.com/Azure/azure-rest-api-specs/blob/master/arm-network/2017-03-01/swagger/applicationGateway.json#L1318), disabledRuleGroups is an array of objects. It wouldn't be user-friendly to ask user to pass those groups as a JSON string, so we need to find a better solution.

## Writing hybrid solution

First of all, we need to remove `disabled-rule-groups` option from generated commands (including related logic). WAF config is an object accessible through `applicationGateway` object, so it's going to be easy to add custom objects to `disabledRuleGroups` array. To do that we need to create a new subcategory for `waf-config` that will control disabled rule groups.

WAF config is a part of `network` service, it's core module code is located in `network._js`. It's not going to be re-generated, so that's where new handwritten commands should be added. In our case, we create new category somewhere in `network._js`.

```javascript
// appGateway is 'application-gateway' category, it was already created
var webApplicationFirewallConfiguration = appGateway.category('waf-config')
  .description($('Commands to manage web application firewall configuration'));
var disabledRuleGroups = webApplicationFirewallConfiguration.category('disabled-rule-groups')
  .description($('Commands to manage disabled rule groups'));
```

Now we write our own command in `disabled-rule-groups` category. The following code is `create` command that adds new disabled rule group to `disabledRuleGroups` array in WAF config. It's options are resource group name and gateway name (you need them to get WAF Config itself), rule group name and comma-separated list of rule IDs. Most of this code can be copypasted from other commands.

```javascript
disabledRuleGroups.command('create [resource-group] [gateway-name] [rule-group-name]')
  .description($('Create a group of disabled rules'))
  .usage('[options] <resource-group> <gateway-name> <rule-group-name>')
  .option('-g, --resource-group <resource-group>', $('the name of the resource group'))
  .option('-w, --gateway-name <gateway-name>', $('the gateway name'))
  .option('-n, --rule-group-name <rule-group-name>', $('the name of the rule group that will be disabled'))
  .option('-r, --rules [rules]', $('The list of rules that will be disabled. If null, all rules of the rule group will be disabled'))
  .execute(function (resourceGroup, gatewayName, ruleGroupName, options, _) {
    resourceGroup = cli.interaction.promptIfNotGiven($('Resource group: '), resourceGroup, _);
    gatewayName = cli.interaction.promptIfNotGiven($('Application gateway name: '), gatewayName, _);
    options.ruleGroupName = cli.interaction.promptIfNotGiven($('rule group name: '), ruleGroupName, _);

    var subscription = profile.current.getSubscription(options.subscription);
    var networkManagementClient = utils.createNetworkManagementClient(subscription);
    var appGateway = new AppGateway(cli, networkManagementClient);
    appGateway.createDisabledRuleGroup(resourceGroup, gatewayName, options, _);
  });
```

AppGateway is handwritten utility for working with application gateways that contains useful reusable code. Method `createDisabledRuleGroup` gets application gateway object, finds WAF config, pushes rule group to `disabledRuleGroups` array and sends updated config to the server through `networkManagementClient`. With some omissions it looks like this.

```javascript
appGateway = networkManagementClient.applicationGateways.get(resourceGroup, appGatewayName, null, _);
var wafConfig = appGateway.webApplicationFirewallConfiguration;
// ...
wafConfig.disabledRuleGroups.push({
  ruleGroupName: options.ruleGroupName,
  rules: options.rules ? options.rules.split(',').map(utils.parseInt) : null
});
// ...
networkManagementClient.applicationGateways.createOrUpdate(resourceGroup, appGatewayName, result, _);
```

Once again, some parts of this code you can see in auto generated commands.

Operations `set`, `show`, `list` and `delete` can be written the same way.
