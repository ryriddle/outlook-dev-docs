---
title: Add and remove attachments in an Outlook add-in
description: You can use the addFileAttachmentAsync and addItemAttachmentAsync methods to attach a file and an Outlook item respectively to the item that the user is composing.
author: jasonjoh
ms.topic: article
ms.technology: office-add-ins
ms.date: 07/14/2017
ms.author: jasonjoh
localization_priority: Normal
---

# Add and remove attachments to an item in a compose form in Outlook

You can use the [addFileAttachmentAsync](https://docs.microsoft.com/office/dev/add-ins/reference/objectmodel/requirement-set-1.5/Office.context.mailbox.item#addfileattachmentasyncuri-attachmentname-options-callback) and [addItemAttachmentAsync](https://docs.microsoft.com/office/dev/add-ins/reference/objectmodel/requirement-set-1.5/Office.context.mailbox.item#additemattachmentasyncitemid-attachmentname-options-callback) methods to attach a file and an Outlook item respectively to the item that the user is composing. Both are asynchronous methods, which means execution can go on without waiting for the action to complete. Depending on the original location and size of the attachment being added, the asynchronous call may take a while to complete.

If there are tasks that depend on the action to complete, you should carry out those tasks in a callback method. This callback method is optional and is invoked when the uploading of the attachment is complete. The callback method takes an [AsyncResult](https://docs.microsoft.com/javascript/api/office/office.asyncresult) object as an output parameter that provides any status, error, and returned value from adding the attachment. If the callback requires any extra parameters, you can specify them in the optional `options.aysncContext` parameter. `options.asyncContext` can be of any type that your callback method expects.

For example, you can define `options.asyncContext` as a JSON object that contains one or more key-value pairs. You can find more examples about passing optional parameters to asynchronous methods in the Office Add-ins platform in [Asynchronous programming in Office Add-ins](https://docs.microsoft.com/office/dev/add-ins/develop/asynchronous-programming-in-office-add-ins#passing-optional-parameters-to-asynchronous-methods). The following example shows how to use the `asyncContext` parameter to pass 2 arguments to a callback method: 

```js
var options = { asyncContext: { var1: 1, var2: 2} };

Office.context.mailbox.item.addFileAttachmentAsync('https://contoso.com/rtm/icon.png', 'icon.png', options, callback);
```

You can check for success or error of an asynchronous method call in the callback method using the `status` and `error` properties of the `AsyncResult` object. If the attaching completes successfully, you can use the `AsyncResult.value` property to get the attachment ID. The attachment ID is an integer which you can subsequently use to remove the attachment.

> [!NOTE]
> As a best practice, you should use the attachment ID to remove an attachment only if the same add-in has added that attachment in the same session. In Outlook on the web and OWA for Devices, the attachment ID is valid only within the same session. A session is over when the user closes the add-in, or if the user starts composing in an inline form and subsequently pops out the inline form to continue in a separate window.

## Attaching a file

You can attach a file to a message or appointment in a compose form by using the `addFileAttachmentAsync` method and specifying the URI of the file. If the file is protected, you can include an appropriate identity or authentication token as a URI query string parameter. Exchange will make a call to the URI to get the attachment, and the web service which protects the file will need to use the token as a means of authentication.

The following JavaScript example is a compose add-in that attaches a file, picture.png, from a web server to the message or appointment being composed. The callback method takes `asyncResult` as a parameter, checks for the result status, and gets the attachment ID if the method succeeds.

```js
Office.initialize = function () {
    // Checks for the DOM to load using the jQuery ready function.
    $(document).ready(function () {
        // After the DOM is loaded, app-specific code can run.
        // Add the specified file attachment to the item
        // being composed.
        // When the attachment finishes uploading, the
        // callback method is invoked and gets the attachment ID. 
        // You can optionally pass any object that you would  
        // access in the callback method as an argument to  
        // the asyncContext parameter.
        Office.context.mailbox.item.addFileAttachmentAsync(
            `https://webserver/picture.png`,
            'picture.png',
            { asyncContext: null },
            function (asyncResult) {
                if (asyncResult.status == Office.AsyncResultStatus.Failed){
                    write(asyncResult.error.message);
                }
                else {
                    // Get the ID of the attached file.
                    var attachmentID = asyncResult.value;
                    write('ID of added attachment: ' + attachmentID);
                }
            });
    });
}

// Writes to a div with id='message' on the page.
function write(message){
    document.getElementById('message').innerText += message;
}
```

## Attaching an Outlook item

You can attach an Outlook item (for example, email, calendar, or contact item) to a message or appointment in a compose form by specifying the Exchange Web Services (EWS) ID of the item and using the `addItemAttachmentAsync` method. You can get the EWS ID of an email, calendar, contact or task item in the user's mailbox by using the [mailbox.makeEwsRequestAsync](https://docs.microsoft.com/office/dev/add-ins/reference/objectmodel/requirement-set-1.5/Office.context.mailbox#makeewsrequestasyncdata-callback-usercontext) method and accessing the EWS operation [FindItem](https://docs.microsoft.com/exchange/client-developer/web-service-reference/finditem-operation). The [item.itemId](https://docs.microsoft.com/office/dev/add-ins/reference/objectmodel/requirement-set-1.5/Office.context.mailbox.item#nullable-itemid-string) property also provides the EWS ID of an existing item in a read form.

The following JavaScript function, `addItemAttachment`, extends the first example above, and adds an item as an attachment to the email or appointment that is being composed. The function takes as an argument the EWS ID of the item that is to be attached. If the attaching succeeds, it gets the attachment ID for further processing, including removing that attachment in the same session.

```js
// Adds the specified item as an attachment to the composed item.
// ID is the EWS ID of the item to be attached.
function addItemAttachment(itemId) {
    // When the attachment finishes uploading, the
    // callback method is invoked. Here, the callback
    // method uses only asyncResult as a parameter,
    // and if the attaching succeeds, gets the attachment ID.
    // You can optionally pass any other object you wish to 
    // access in the callback method as an argument to 
    // the asyncContext parameter.
    Office.context.mailbox.item.addItemAttachmentAsync(
        itemId,
        'Welcome email',
        { asyncContext: null },
        function (asyncResult) {
            if (asyncResult.status == Office.AsyncResultStatus.Failed){
                write(asyncResult.error.message);
            }
            else {
                var attachmentID = asyncResult.value;
                write('ID of added attachment: ' + attachmentID);
            }
        });
}
```

> [!NOTE]
> You can use a compose add-in to attach an instance of a recurring appointment in Outlook on the web or OWA for Devices. However, in a supporting Outlook rich client, attempting to attach an instance would result in attaching the recurring series (the master appointment).

## Removing an attachment

You can remove a file or item attachment from a message or appointment item in a compose form by specifying the corresponding attachment ID and using the [removeAttachmentAsync](https://docs.microsoft.com/office/dev/add-ins/reference/objectmodel/requirement-set-1.5/Office.context.mailbox.item#removeattachmentasyncattachmentid-options-callback) method. You should remove only attachments that the same add-in has added in the same session. You should make sure the attachment ID corresponds to a valid attachment, or the method will return an error. Similar to the `addFileAttachmentAsync` and `addItemAttachmentAsync` methods, `removeAttachmentAsync` is an asynchronous method. You should provide a callback method to check for the status and any error by using the `AsyncResult` output parameter object. You can also pass any additional parameters to the callback method by using the optional `asyncContext` parameter.

The following JavaScript function, `removeAttachment`, continues to extend the examples above, and removes the specified attachment from the email or appointment that is being composed. The function takes as an argument the ID of the attachment to be removed. You can obtain the ID of an attachment after a successful `addFileAttachmentAsync` or `addItemAttachmentAsync` method call, and store it for a subsequent `removeAttachmentAsync` method call.

```js
// Removes the specified attachment from the composed item.
// ID is the Exchange identifier of the attachment to be 
// removed. 
function removeAttachment(attachmentId) {
    // When the attachment is removed, the
    // callback method is invoked. Here, the callback
    // method uses an asyncResult parameter and gets
    // the ID of the removed attachment if the removal
    // succeeds.
    // You can optionally pass any object you wish to 
    // access in the callback method as an argument to 
    // the asyncContext parameter.
    Office.context.mailbox.item.removeAttachmentAsync(
        attachmentId,
        { asyncContext: null },
        function (asyncResult) {
            if (asyncResult.status == Office.AsyncResultStatus.Failed){
                write(asyncResult.error.message);
            }
            else {
                write('Removed attachment with the ID: ' + asyncResult.value);
            }
        });
}
```

## See also

- [Create Outlook add-ins for compose forms](compose-scenario.md)
- [Asynchronous programming in Office Add-ins](https://docs.microsoft.com/office/dev/add-ins/develop/asynchronous-programming-in-office-add-ins)
