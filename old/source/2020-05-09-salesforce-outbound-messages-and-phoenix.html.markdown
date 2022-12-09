---
title: Salesforce Outbound messages and Phoenix
date: 2020-05-09 02:11 UTC
tags: Salesforce, Elixir, Phoenix, XML
---
# Let's sell some stuff!
[Salesforce](https://www.salesforce.com/) is an incredibly popular and powerful sales tracking system. With over 150k customers
and an extensible platform, it's a great opportunity to use our favorite programming language to get in on the action.

There are several libraries to access the SalesForce API like Jeff Weiss' [forcex](https://github.com/jeffweiss/forcex),
but not too much written about handling Outbound Messages.

## Outbound Messages
Outbound Messages are webhooks that are triggered when a Salesforce object is updated. There are 
[tutorials](https://medium.com/@MicroPyramid/how-to-set-up-outbound-messaging-in-salesforce-42a0a08e65b1) 
on how to [setup](https://www.salesforcetutorial.com/generating-outbound-message-workflow-action/)
the Salesforce side of this, so I won't be covering it. Assume there's an Outbound Messages set to trigger whenever
the _Stage_ of an _Opportunity_ has been updated.

### Getting the message
The first thing we need to do is build a way to accept the Outbound message from Salesforce. This may make some of you
cringe but we're going to have to accept... XML. Particularly, a SOAP response. If you don't know what SOAP is, don't 
worry about it. We're going to treat this like regular XML.

### Plug Parsers
Crack open your `endpoint.ex` file and you should see something like this:

```elixir
  plug Plug.Parsers,
    parsers: [:urlencoded, :multipart, :json],
    pass: ["*/*"],
    json_decoder: Phoenix.json_library()
```
Under the covers, Plug comes with a few modules to handle json, multipart and url encoded requests.
You can take a look at the code for `urlencoded`'s parser
[here](https://github.com/elixir-plug/plug/blob/65986ad32f9aaae3be50dc80cbdd19b326578da7/lib/plug/parsers/urlencoded.ex).
Every parser needs 2 functions. an `init/1` function for compiled configuration, and a `parse/5` function.
The [`parse/5`](https://hexdocs.pm/plug/Plug.Parsers.html#c:parse/5) function returns a tuple with `{:ok, params, conn}`
if it _can_ parse this kind of request, or `{:next, conn}` if it can't.

One parser that's missing is an XML parser! No worries. We can handle all of our processing of the Outbound Message in the
parser leaving our controller to code to deal with our nice, cleanly parsed data.

Let's make a new Parser.

```elixir
defmodule SFDCWebhookParser do
  def init(opts), do: opts
  def parse(conn, "text", "xml", _headers, opts) do
    {:ok, body, conn} = Plug.Conn.read_body(conn, opts)
    notifications = extract_from_webhook(body)
    {:ok, %{notifications: notifications}, conn}
  end
  def parse(conn, _type, _subtype, _headers, _opts), do: {:next, conn}
end
```

Here we have a parser that responds to messages with a content type of "text/xml" since we're getting a SOAP message. 
We call `Plug.Conn.read_body/2` in order to load request body. Then we pass it into our (as yet to be written) `extract_from_webhook/1` function.
In order to work on that... we'll need to dive into parsing XML with [SweetXml](https://hexdocs.pm/sweet_xml/SweetXml.html).

### SweetXml
For this next bit, we're going to need to talk about [XPath](https://www.w3schools.com/xml/xpath_intro.asp). XPath is a way of 
traversing nodes in a tree. Take this HTML for instance:

```elixir
html = """
<div>
  <ul edible="no">
    <li>One fish</li>
    <li>Two Fish</li>
    <li>Red Fish</li>
    <li>Blue Fish</li>
  </ul>
</div>
"""
```

Now we can use `SweetXml` to extract out some data from this markup and put in in a map for us.

```elixir
import SweetXml
html
|> xpath(
~x"//ul", # From the root, find a ul node
 edible: ~x"@edible", # From that node, read its `edible` attribute
 items: ~x"./li/text()"l # Also from that node, find any li nodes and return their text. The `l` informs it we want a list back.
)
%{edible: 'no', items: ['One fish', 'Two Fish', 'Red Fish', 'Blue Fish']}
```
With this, we have enough to get data out of our Webhook.

### Notification Message
The [docs](https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/sforce_api_om_outboundmessaging_wsdl.htm) give
an outline of the anatomy of a message. Here's a sample message from the sandbox environment:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
 <soapenv:Body>
  <notifications xmlns="http://soap.sforce.com/2005/09/outbound">
   <OrganizationId>00D5w000004qGTOEA2</OrganizationId>
   <ActionId>04k5w000000TSwMAAW</ActionId>
   <SessionId>...</SessionId>
   <EnterpriseUrl>...</EnterpriseUrl>
   <PartnerUrl>...</PartnerUrl>
   <Notification>
    <Id>04l5w00005286hAAAQ</Id>
    <sObject xsi:type="sf:Opportunity" xmlns:sf="urn:sobject.enterprise.soap.sforce.com">
     <sf:Id>0065w000023STAmAAO</sf:Id>
     <sf:AccountId>0015w00002BFDeLAAX</sf:AccountId>
     <sf:Amount>45000.0</sf:Amount>
     <sf:CloseDate>2020-07-11</sf:CloseDate>
     <sf:ContactId>0035w000035gg9GAAQ</sf:ContactId>
     <sf:CreatedById>0055w00000BhqR0AAJ</sf:CreatedById>
     <sf:CreatedDate>2020-05-08T13:40:22.000Z</sf:CreatedDate>
     <sf:FiscalQuarter>3</sf:FiscalQuarter>
     <sf:FiscalYear>2020</sf:FiscalYear>
     <sf:Follow_Up__c>false</sf:Follow_Up__c>
     <sf:HasOpenActivity>false</sf:HasOpenActivity>
     <sf:HasOpportunityLineItem>false</sf:HasOpportunityLineItem>
     <sf:HasOverdueTask>false</sf:HasOverdueTask>
     <sf:IsClosed>false</sf:IsClosed>
     <sf:IsDeleted>false</sf:IsDeleted>
     <sf:IsWon>false</sf:IsWon>
     <sf:LastModifiedById>0055w00000BhqR0AAJ</sf:LastModifiedById>
     <sf:LastModifiedDate>2020-05-09T03:04:32.000Z</sf:LastModifiedDate>
     <sf:LastReferencedDate>2020-05-09T03:06:37.000Z</sf:LastReferencedDate>
     <sf:LastViewedDate>2020-05-09T03:06:37.000Z</sf:LastViewedDate>
     <sf:LeadSource>External Referral</sf:LeadSource>
     <sf:Name>Backpackers, Inc. (Sample)</sf:Name>
     <sf:OwnerId>0055w00000BhqR0AAJ</sf:OwnerId>
     <sf:Probability>80.0</sf:Probability>
     <sf:StageName>Negotiation/Review</sf:StageName>
     <sf:SystemModstamp>2020-05-09T03:04:32.000Z</sf:SystemModstamp>
    </sObject>
   </Notification>
  </notifications>
 </soapenv:Body> 
</soapenv:Envelope>
```
It's pretty beefy, but parsing will be pretty straight forward. The documentation states we may get up to 100 notification 
nodes.

```elixir
import SweetXml
def extract_from_webhook(body) do
  body |> xpath(~x"//Notification"l, # Support for multiple notifications
  message_id: ~x"./Id/text()",
  type: ~x"./sObject/@xsi:type",
  object_id: ~x"./sObject/sf:Id/text()",
  account_id: ~x"./sObject/sf:AccountId/text()",
  amount: ~x"./sObject/sf:Amount/text()",
  close_date: ~x"./sObject/sf:CloseDate/text()",
  contact_id: ~x"./sObject/sf:ContactId/text()",
  created_by_id: ~x"./sObject/sf:CreatedById/text()",
  created_date: ~x"./sObject/sf:CreatedDate/text()",
  fiscal_quarter: ~x"./sObject/sf:FiscalQuarter/text()",
  fiscal_year: ~x"./sObject/sf:FiscalYear/text()",
  follow_up: ~x"./sObject/sf:Follow_Up__c/text()",
  has_open_activity: ~x"./sObject/sf:HasOpenActivity/text()",
  has_opportunity_line_item: ~x"./sObject/sf:HasOpportunityLineItem/text()",
  has_overdue_task: ~x"./sObject/sf:HasOverdueTask/text()",
  is_closed: ~x"./sObject/sf:IsClosed/text()",
  is_deleted: ~x"./sObject/sf:IsDeleted/text()",
  is_won: ~x"./sObject/sf:IsWon/text()",
  last_modified: ~x"./sObject/sf:LastModifiedById/text()",
  last_modified_date: ~x"./sObject/sf:LastModifiedDate/text()",
  last_reference_date: ~x"./sObject/sf:LastReferencedDate/text()",
  last_viewed_date: ~x"./sObject/sf:LastViewedDate/text()",
  lead_source: ~x"./sObject/sf:LeadSource/text()",
  name: ~x"./sObject/sf:Name/text()",
  owner_id: ~x"./sObject/sf:OwnerId/text()",
  probability: ~x"./sObject/sf:Probability/text()",
  stage_name: ~x"./sObject/sf:StageName/text()",
  system_modstamp: ~x"./sObject/sf:SystemModstamp/text()"
)
end
```
This will return the following as params:

```elixir
[
  %{
    account_id: '0015w00002BFDeLAAX',
    amount: '45000.0',
    close_date: '2020-07-11',
    contact_id: '0035w000035gg9GAAQ',
    created_by_id: '0055w00000BhqR0AAJ',
    created_date: '2020-05-08T13:40:22.000Z',
    fiscal_quarter: '3',
    fiscal_year: '2020',
    follow_up: 'false',
    has_open_activity: 'false',
    has_opportunity_line_item: 'false',
    has_overdue_task: 'false',
    is_closed: 'false',
    is_deleted: 'false',
    is_won: 'false',
    last_modified: '0055w00000BhqR0AAJ',
    last_modified_date: '2020-05-09T03:04:32.000Z',
    last_reference_date: '2020-05-09T03:06:37.000Z',
    last_viewed_date: '2020-05-09T03:06:37.000Z',
    lead_source: 'External Referral',
    message_id: '04l5w00005286hAAAQ',
    name: 'Backpackers, Inc. (Sample)',
    object_id: '0065w000023STAmAAO',
    owner_id: '0055w00000BhqR0AAJ',
    probability: '80.0',
    stage_name: 'Negotiation/Review',
    system_modstamp: '2020-05-09T03:04:32.000Z',
    type: 'sf:Opportunity'
  }
]
```

Woot! Now that map will be passed in `params` to your controller actions. There's one more thing we need to make Salesforce happy.

### The response.

Say we're routing Outbound messages to `SalesforceWeb.WebhookController.webhook/2`. We'll need to respond with a SOAP
response to acknowledge the message.

```elixir
defmodule SalesforceWeb.WebhookController do
  use SalesforceDotComWeb, :controller

  def webhook(conn, params) do
    params |> do_cool_things()
    conn
    |> put_resp_content_type("text/xml")
    |> send_resp(200, acknowledgement())
  end

  def acknowledgement do
    """
    <?xml version="1.0" encoding="UTF-8"?>
        <soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/">
            <soapenv:Body>
                <notificationsResponse xmlns="http://soap.sforce.com/2005/09/outbound">
                    <Ack>true</Ack>
               </notificationsResponse>
            </soapenv:Body>
        </soapenv:Envelope>
    """ |> String.trim()
  end
end
```

With this, we're all set! NOTE: The acknowledgement will always be the same.

# Next Steps
It'd be nice to have type conversion of strings to booleans and dates, but this is a great start. I hope this was informative.

Happy clacking!
