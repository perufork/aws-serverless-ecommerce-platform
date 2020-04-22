Testing strategies
==================

This projects use three different layers of tests to validate that the different services are behaving as expected. __Unit tests__ validate that the code within AWS Lambda functions is working properly, __integration tests__ validate that a specific microservice honors its contracts, and __end-to-end tests__ simulate the entire flow to see that core functionalities from the user perspectives are working as expected.

These multiple layers of tests also helps with ensuring that individual tests are working as expected. As test code is code, it is also prone to bugs and errors.

## End-to-end tests

<p align="center">
  <img src="images/flow.png" alt="User flow"/>
</p>

The __end-to-end tests__ mimick the steps that customers and internal staff would go through to fulfill an order. They only use the external APIs to both act and validate that their actions resulted in the right outcome. For example, a customer creating a valid order should result in a packaging request in the _warehouse_ service's API to retrieve incoming packaging requests. You can find those tests [in the shared folder](../shared/tests/e2e/).

As they look at the entire application as a black box, they do not necessarily help with finding the root cause of an issue. In the previous case, the failure could reside in the _orders_ or in the _warehouse_ service.

As part of the deployment process, they work well to sanity-check that the entire system is behaving as expected, but multiple parallel deployment from different services could make one service pipeline fail because of an issue in another service. For that reason, they are run _after_ the integration tests.

## Integration tests

<p align="center">
  <img src="../delivery/images/delivery.png" alt="Delivery architecture diagram" />
</p>

The purpose of the __integration tests__ is to test that the promises of the service are being met. For example, if a service needs to react to a specific event, these tests will send that event onto the event bus and compare the results with the desired outcome. These tests are bounded to a given service, and thus do not serve the purpose of testing integration between services (which is done by end-to-end tests).

For example, when the _warehouse_ service creates a package, it emits an event to EventBridge. The _delivery_ service should pick that event up and create an item in DynamoDB. Therefore, an integration test will create that event and look if the item was properly created in DynamoDB.

This type of test has the benefit of validating that the resources are configured properly, such as making sure that the Lambda function is triggered by events from EventBridge, that it has the right permissions to access the DynamoDB table, etc. in an actual AWS environment.

However, one downside of this approach is that, because the _delivery_ service fakes events from the _warehouse_ service, it has to rely that the _warehouse_ service's documentation is correct. If the event when a package is created is slightly different and doesn't match the rule, the integration tests could pass, but the service would never receive those events. However, the end-to-end tests help catch these types of issues.

On the integration with other services, there are two possible strategies: either services provide mocks (APIs and way to generate events), or services are called during the integration tests of a given service. This project uses the latter option for simplicity and to reduce the overhead of keeping the mocks in sync.

This means that integration tests for one service could be tangled to another service's architecture. For example, when testing the create order path of the _orders_ service, it will make calls to the _products_ service to validate that the products in the order request exist. Therefore, the tests need to inject data into the _products_ database. If the way to inject those products changes, the test will fail despite the fact that the functionality itself might be working properly. This thus requires adapting tests due to external changes that do not affect the functionalities in scope of the test.

## Unit tests

Many functional aspects of a service are already tested by the end-to-end and integration tests. __Unit tests__ provide a few additional benefits and are complementary to the other types of test.

First, they are much quicker to run as they do not need to deploy or update resources on AWS. This means that unit tests can potentially explore more scenarios in a given timeframe than the other types of test.

Given the fact that they test code locally, code coverage tools can be used to ensure that they cover all branches and functions for all Lambda functions within a service. For this reason, we enforce a test coverage of [at least 90%](../shared/tests/unit/coveragerc).

Finally, as a service should have multiple layers of security controls to prevent against misconfiguration, unit tests could test code branches that are not used in normal operation. For example, if a function behind an API Gateway should only be called with IAM credentials, the API Gateway will normally reject requests without a proper authorization header. However, as a best practice, the Lambda function code should check if IAM credentials are present. This section of code will never be used in normal operations, but still needs to be present to protect against configuration mishaps.

## Lint

To enforce a certain quality standard and set of stylistic rules across the entire project, lint tools analyze the code for CloudFormation templates, OpenAPI documents and Lambda function source code.

For CloudFormation templates, there are [additional rules](../shared/lint/rules/) specific to this project, such as enforcing that all Lambda function have a corresponding CloudWatch Logs log group, or that functions called asynchronously have a [destination](https://aws.amazon.com/blogs/compute/introducing-aws-lambda-destinations/) in case of failures to process the messages.

## Running tests

When using the _ci_ or _all_ command on a service (e.g. `make ci-$SERVICE` or `make all-$SERVICE`), this will automatically run the linting, unit tests for _ci_ and linting, unit and integration tests for _all_. However, if you want to run the tests manually, you can use the following commands:

* `make lint-$SERVICE`
* `make tests-unit-$SERVICE`
* `make tests-integ-$SERVICE`
* `make tests-e2e`

__Remark__: As end-to-end tests look at the flow across services, you cannot run these for a specific service, only for the entire platform. This means that all services should be deployed for those tests to work properly.

## Writing tests

_TODO_

## Helpers and fixtures

_TODO_