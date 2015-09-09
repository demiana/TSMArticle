## API Design - A Case Study##

Nowadays it’s getting harder and harder for a closed product/system to be successful and thus the need to expose an API. API (acronym for Application Programming Interface) clearly defines how the rest of the software world will interact with a software system (or component). An API brings positive advantages like scaling market reach, reduced time to market, flexibility and innovation. Without APIs we could argue that none of the important Web players like Microsoft, Google, Facebook or Twitter (just to name a few) would be as big as they are today.

### Why is API design so hard? ###
Building an API is one of the most important things that can be done to increase the value of a service, however designing a great API is challenging. A well-designed API should reflect the goals of the business it is designed to serve while at the same time taking into account the organization’s specific strengths and limitations in terms of budget, personal skill sets and technical infrastructure. The key is to know what the API needs to accomplish, unfortunately this changes over time because the business changes. We all know how lack of flexibility and changing requirements can lead to unmaintainable 'spaghetti' code, in the API world the same causes lead to methods/operations overloading, legacy versions and deprecation. It follows that the main quality of an API is to support the business and to be flexible enough to survive business changes. Other aspects that should be taken into account when designing or evaluating APIs are:

- ease of learning, extension and integration in the client application
- likelihood and consequences of misuse
- impact over the client code, in terms of maintenance costs
- the level at which is satisfies the client’s business requirements
- appropriateness for the target audience

Following the above guidelines will ensure that the consumers of an API will be able to understand it. Moreover, integration and usage will become quite easy. The easier the API is to consume, the higher the adoption rates will be and the lower the integration costs.

###The power of example###

Let's put the above considerations in practice and analyze a real-world API together with some of the challenges we faced throughout the API design process. The following examples are taken from a web service whose aim is to manage the promotions visible on our website. These promotions ask the customer to perform multiple type of actions on the site (such as registering, placing bets, playing arcade games), in order to receive one of the pre-configured type of rewards.

In our organization, web services communicate with each other using an RPC-style protocol. In order to describe the operations supported by each service, we use a bespoke XML schema called BSIDL (acronym for Betfair Service Interface Definition Language). The BSIDL specifications are in XML format and describe the operations (or methods) the service exposes and the data types that are supported. Below is an example of the operation used by one of the APIs clients (the back office tool) to create a promotion.
```xml
<operation name="create">
       <description>Creates a new promotion.</description>
       <parameters>
           <request>
               <parameter name="promotion" type="Promotion" mandatory="true">
                   <description>The promotion to create.</description>
               </parameter>
           </request>
           <response type="ResponseStatus">
               <description>Return ok if it was a successful call and fail otherwise, together with an error code.
			</description>
           </response>
       </parameters>
</operation>	
```
It can be seen that the operation name is explicit, basically the name says what the operation does and is hard to misuse having one parameter whose name is suggestively named `promotion`(expresses what we want to create) but it is powerful enough to satisfy the requirements. Up to this point we have an intuitive to use and self documenting operation.

The main concern regarding this new interface was how to model a promotion such that it is easy to extend in the future, but at the same time avoids duplication as much as possible. The simplest and easiest way could have been to create a new type for each promotion, but this choice would have lead to a number of disadvantages like duplicate fields - promotions have in common a series of characteristics like name, description; the final result will be an explosion of promotion types for each new kind of promotion that must be added. We were not fully happy with this approach and after a bit of thinking we came up with the idea of a promotion type which has all the fields necessary for describing a promotion with a special field called `fulfillmentCriteria`.  An excerpt of the promotion type can be seen below.
```xml
    <dataType name="Promotion">
       <description>Encapsulates the promotion entity fields.</description>
       <parameter name="name" type="string">
           <description>The name of the promotion.</description>
       </parameter>
   	<parameter name="rank" type="i32">
   		<description>Represents the promotion ranking number which is an integer on range[1,1000].
   		</description>
   	<parameter name="product" type="set(Product)">
           <description>
               Indicates the products for which the promotion is available.
           </description>
       </parameter>
       <parameter name="fulfillmentCriteria" type="FulfillmentCriteria">
           <description>What does the user need to accomplish in order to get the reward.
   		</description>
   	</parameter>
   </dataType>	
```
The solution to the multiple types of promotion problem was to use composition at the API definition level.  For example, instead of creating a new promotion type for promotions that require a deposit (e.g. deposit 10 EUR to get a free bet) we created a new criteria called `DepositCriteria` which encapsulates the 10 EUR deposit requirement. This makes the API easy to extend without any breaking changes when adding new types of `FulfillmentCriteria` with the advantage of having a single promotion type.
```xml
<dataType name="FulfillmentCriteria">
	<parameter name="compoundCriteria" type="CompoundCriteria"/>
	<parameter name="depositCriteria" type="DepositCriteria"/>
    <parameter name="placeBetCriteria" type="PlaceBetCriteria"/>
	<parameter name="registerCriteria" type="RegisterCriteria"/>
</dataType>
```
The addition of the `CompoundCriteria` was necessary for the cases when a promotion would have been configured to have multiple criteria in the same time using operator types `AND`,`OR` describing what the Betfair customer needs to do. This is how the  `CompoundCriteria` tree looks like if we will have a promotion with the following requirements:
> Register now then play 5 EUR on Arcade games or place a bet of 5 on Romania vs Hungaria to get a free bet.

![](https://drive.google.com/uc?export=&id=0B8bE-oPfVra5TS1QX3l2Rm5VNXc)

Regarding the error handling, it can be seen from the first example that the `create` operation returns a data type called `ResponseStatus`. The promotion is validated at creation time and all inconsistencies are encapsulated and reported in the returned `ResponseStatus` object. We approached this solution because we think is more appropriate and consumer friendly and as Martin Fowler said:
> If a failure is expected behavior, then you shouldn't be using exceptions.

The client in this case verifies the `success` flag and only if the response is negative checks the `errorCode`, otherwise does nothing. As simple as that. Based on the error code received the client can decide what to do.
```xml
<dataType name="ResponseStatus">
   <parameter name="success" type="bool"/>
   <parameter name="errorCode" type="string"/>
</dataType>
```
Even when the guidelines are followed when designing an API mistakes can be made. New requirements arrive all the time therefore new attributes describing the promotion were added.
```java
Promotion
  getPromotionName()
  getRank()
  getProduct()
  getDisplayAttributes()
  getJurisdiction()
  getVisibleForGuest()
  getTermsAndConditions()
```
Some of the attributes were grouped under new data types with suggestive names.

    DisplayAttributes
       getTheme()
       getBannerUrl()
       getImageUrl()	
			
Others were added directly without being grouped because we didn't find at that moment a suitable group and adding a single attribute in a group would not make sense. Now, as a consequence promotion type attributes grew a lot and even if we found a possible group for some of the attributes (we could group `rank` and `product` into something similar to `DisplayAttributes`), API client already uses this version and is quite hard to make breaking changes.

### Conclusions ###
Designing an API is a challenging activity. What we have learned from this experience is that even if not all the requirement changes can be predicted all the time, it is a good practice to try to anticipate the future specifications because, in this way the API will be easier to extend and there will be less breaking changes. Another lesson learned is that doing things right from the first time it not always possible, but is important to be able to recognize and fix  problems as soon as possible.