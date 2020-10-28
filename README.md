# CreateInvoiceandCollectPaymentfromCreator
A method to collect payment from a Creator form that acts as a sort of registration or purchase order. It generates a Zoho Invoice record and redirects to its Payment Page (the same page people are sent to when they click the 'Pay Now' link on the Invoice email) upon submission.

## Core Idea

Suppose you had a Registration Form for an event that requires payment but you wanted to accept registrations on deferred payment terms like a standard invoice. Or imagine it was an order form that allowed customers to place orders and pay on terms. Rather than *requiring* immediate payment to complete registration (like a typical 'payment portal' on a form) you could generate an invoice and 'encourage' immediate payment by redirecting the form submitter to the invoice payment page when they submit the form. This script does just that--upon submission it:

A: Creates a Zoho Invoice Customer (if necessary)
B: Creates a Zoho Invoice record for the customer related to the form submission
C: Redirects the form submitter to the Invoice Payment Page to collect immediate payment.

## Related Zoho Apps:

Zoho Creator &
Zoho Invoice*

*This could also work with Zoho Books with slight modification.
*Same code could work in Zoho CRM and other systems with slight modification with something like a Custom Button.

## The Process In-Depth

A: In order to create an Invoice for this form submission, we need to have a Customer record in Zoho Invoice for the submitter. We may already have a Customer record or we may not. If we don't, we need to make one. If we do, we need to know so we can associate an Invoice to that existing customer. That's what this script does. (see script below with comments).

```javascript
// We search the Contacts module in Zoho Invoice for Contacts with a matching email address to the one provided.
searchParam = {"email":email};
contactSearch = invokeurl
  [
	 url :"https://invoice.zoho.com/api/v3/contacts"
	 type :GET
	 parameters:searchParam
	 connection:"zohoinvoice"
  ];

// Check If Contact is Found (If 'contactSearch' which is a List variable, is not empty) and query the primary contact's info to put on the invoice
if(contactSearch.get("contacts").size() > 0)
		{
			// Get Contact Record
			contact = contactSearch.get("contacts").get(0);
			contactId = contact.get("contact_id");
			contactDetails = zoho.invoice.getRecordById("contacts",orgId,contactId.toLong()).get("contact");
			contactPersons = contactDetails.get("contact_persons").toList();
			// Find Primary Contact
			for each  contactPerson in contactPersons
			{
				if(contactPerson.get("is_primary_contact"))
				{
					contactPersonId = contactPerson.get("contact_person_id");
				}
			}
		}
 // If the search function returned nothing then we know we need to make a Contact which we do below. Note: The Variables here like firstname and lastname and whatnot come from the form.
 else
		{
			// Create Map for New Contact
			newContactMap = Map();
			newContactMap.put("email",email);
			newContactMap.put("contact_name",firstName + " " + lastName);
			newContactMap.put("contact_persons",{{"first_name":firstName,"last_name":lastName,"email":email}});
			// Create New Contact
			createContact = zoho.invoice.create("Contacts",orgId,newContactMap);
			contact = createContact.get("contact");
			contactId = contact.get("contact_id");
			// Get Contact Person Info
			contactPerson = contact.get("contact_persons").get(0);
			contactPersonId = contactPerson.get("contact_person_id");
		}
```

B: Once we have our Zoho Invoice customer, we can put together our Invoice. 

```javascript
// Create New Invoice map and select a specific template if needed
		invoice = Map();
		invoice.put("customer_id",contactId);
		invoice.put("date",today);
		//invoice.put("template_id","your_templateId");
		contactPersonsList = List();
		contactPersonsList.add(contactPersonId);
		invoice.put("contact_persons",contactPersonsList);
    //Line items are added as a map. Here we have a single line item with its ID and a custom Fee from the form.
		invoice.put("line_items",{{"item_id":"2062836000000758051","quantity":"1","rate":input.Fee}});
		// Create List of Custom Fields
		customFieldList = List();
		customFieldMap1 = Map();
		customFieldMap1.put('label','Custom Field 1');
		customFieldMap1.put('value',customformfieldvalue);
		customFieldList.add(customFieldMap1);
		// // for second custom field
		// customField2 = Map();
		// customField2.put("label","your_2nd_label");
		// customField2.put("value",ifnull(your_2nd_value,""));
		// customList.add(customField2);
		invoice.put("custom_fields",customFieldList);
		// Set Payment Options - this code enables both Paypal and Stripe, but you can of course customize it
		paymentOptions = List();
		paypalMap = Map();
		paypalMap.put("gateway_name","paypal");
		paypalMap.put("additional_field1","standard");
		//stripeMap = Map();
		//stripeMap.put("gateway_name","stripe");
		paymentOptions.add(paypalMap);
		//paymentOptions.add(stripeMap);
		gatewayMap = Map();
		gatewayMap.put("payment_gateways",paymentOptions);
		invoice.put("payment_options",gatewayMap);
		// Create Invoice if the discount is not 100%
		createInvoice = zoho.books.createRecord("invoices",orgId,invoice);
		info createInvoice;
```
```javascript
C: Once we've created the Invoice we can get its Payment Link out and redirect the form submitter to that page.

//First we mark it as Sent. This enables us to redirect them to the Payment Link later.
marksent = invokeurl
  [
	url :"https://invoice.zoho.com/api/v3/invoices/" + createInvoice.get("invoice").get("invoice_id") + "/status/sent"
	type :POST
	connection:"zohoinvoice"
  ];
//Get the Payment Link off of the response variable from the Invoice creation above and redirect!
paymentlink = createInvoice.get("invoice").get("invoice_url");
openUrl(paymentlink,"parent window");
```
