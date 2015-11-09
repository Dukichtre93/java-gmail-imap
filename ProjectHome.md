# What's new? #

`JavaMail` 1.4.6 now contains experimental support for Gmail extensions.

[Announcing EXPERIMENTAL Gmail features support in JavaMail](http://kenai.com/projects/javamail/lists/users/archive/2012-08/message/0)

The code isn't too dissimilar from the java-gmail-imap code so migration back to official `JavaMail` should be relatively painless.

[See my initial JavaMail 1.4.6/Gmail experiment](http://code.google.com/p/java-gmail-imap/wiki/TestingJavaMailGIMAP)

**Many thanks to everyone who contributed code patches and feedback to this project, it has been a pleasure working with you**


---


For developers on Google Apps for Business/Education domains I have added a page describing how the equivalent of 2-legged OAuth 1.0a can be achieved using OAuth 2.0.

[How to achieve domain-wide delegation of authority on Gmail with OAuth 2.0](http://code.google.com/p/java-gmail-imap/wiki/GmailAndXOAUTH2)

Thanks to user feedback I now have an improved thread gathering method - [Displaying Gmail Thread Based View - Improved](http://code.google.com/p/java-gmail-imap/wiki/DisplayingGmailThreadBasedViewImproved)

Version 0.5 adds support for the [XLIST command](https://developers.google.com/google-apps/gmail/imap_extensions#extension_of_the_list_command_xlist) which populates the returned folders list with additional attributes.  These can be used to map localized folder names to the special folders equivalents (e.g. the **Todos** folder in Spanish is identified as **`AllMail`**).  The current list of special folders is: Inbox, Starred, Sent Items, Draft, Spam, All Mail. [See XLIST example.](http://code.google.com/p/java-gmail-imap/wiki/XLISTExample)



**An extension of `JavaMail` 1.4.4 to support the GMail IMAP protocol extensions**

Gmail supports IMAP but has also provided some IMAP extensions for developer use.

[Gmail IMAP Extensions](https://developers.google.com/google-apps/gmail/imap_extensions)

This project extends `JavaMail 1.4.4` to provide additional support for some of these extensions.  This makes it possible to present the user with a thread based view of their e-mails (as GMail does).

IMAPMessage gains several new methods including:

```
long getGoogleMessageId();
long getGoogleMessageThreadId();
String[] getGoogleMessageLabels();
```


---


**How to make use of this**

A copy of the source code of `JavaMail` 1.4.4 has been made and the package structure has been relocated to the following packages - to avoid confusion:

  * com.google.code.com.sun.mail
  * com.google.code.javax.mail

So this is not a drop in replacement for `JavaMail` 1.4.4 but it should mostly work in the same way.

If you are interested in what changes were necessary to `JavaMail` 1.4.4 to enable support for the GMail IMAP protocol extensions then please look [at the source code repository under the 0.1 tag](http://code.google.com/p/java-gmail-imap/source/browse/#svn%2Ftags%2F0.1%2Fsrc%2Fmain%2Fjava).

One possible use of this code is when accessing Gmail accounts using oAuth - see: [google-mail-xoauth-tools](http://code.google.com/p/google-mail-xoauth-tools/wiki/JavaSampleCode).

Once you have obtained your IMAPSSLStore you can use the `FetchProfile` mechanism to use the IMAP extension data that is made available by the code in this project:


**Example: Obtaining X\_GM\_MSGID, X\_GM\_THRID and X\_GM\_LABELS values for messages:**
```
...
        IMAPSSLStore store = ...
        Folder inbox = null;
        try {
            inbox = store.getFolder("INBOX");
            inbox.open(Folder.READ_ONLY);
            Message[] ms = inbox.getMessages();

            FetchProfile fp = new FetchProfile();
            fp.add(IMAPFolder.FetchProfileItem.X_GM_MSGID);
            fp.add(IMAPFolder.FetchProfileItem.X_GM_THRID);
            fp.add(IMAPFolder.FetchProfileItem.X_GM_LABELS);
            inbox.fetch(ms, fp);

            for (Message m : ms) {
                IMAPMessage im = (IMAPMessage) m;
                System.out.println(im.getGoogleMessageId());
                // Hex version - useful for linking to Gmail
                System.out.println(Long.toHexString(im.getGoogleMessageId()));
                System.out.println(im.getGoogleMessageThreadId());
                String[] labels = im.getGoogleMessageLabels();
                if(labels!=null){
                    for(String label: labels){
                        System.out.println("Label: " + label);
                    }
                }
            }
        } finally {
            if (inbox.isOpen()) {
               inbox.close(true);
            }
            store.close();
        }
...
```

**Searching Support:** The `SearchTerm` implementation allows for searching GMail by  message id (X-GM-MSGID), thread Id (X-GM-THRID), label (X-GM-LABELS) and using the [advanced search syntax](http://mail.google.com/support/bin/answer.py?answer=7190) (X-GM-RAW).  (Although `SearchTerm` using X-GM-RAW is not fully implemented)

**Example: Searching for messages in the same thread:**
```
...

public static void simpleSearch(IMAPSSLStore store) throws MessagingException {
    FetchProfile fp = new FetchProfile();
    fp.add(FetchProfile.Item.ENVELOPE);
    fp.add(IMAPFolder.FetchProfileItem.X_GM_THRID);
    IMAPFolder folder = null;
    try {
        folder = (IMAPFolder) store.getFolder("INBOX");
        if (folder != null) {
            folder.open(Folder.READ_ONLY);
            int msgCount = folder.getMessageCount();
            IMAPMessage lastMsg = (IMAPMessage) folder.getMessage(msgCount);
            folder.fetch(new Message[]{lastMsg}, fp);
            long threadId = lastMsg.getGoogleMessageThreadId();
            
            System.out.println(threadId);
            
            GmailThreadIDTerm term = new GmailThreadIDTerm(threadId + "");
            Message[] thread = folder.search(term);
            folder.fetch(thread, fp);
            
            for(Message m: thread){
                System.out.println("### " + m.getSubject());
            }
            
        }
    } finally {
        if (folder.isOpen()) {
            folder.close(true);
        }
        store.close();
    }
}

...
```

The following is an example transcript of a call to SEARCH using the X-GM-THRID attribute, which could result by running the above Java method:

```
A5 SEARCH X-GM-THRID 1375670460152672467 ALL
* SEARCH 8 9 10 12
A5 OK SEARCH completed (Success)
```


---


## Possible future directions ##

  * **Compression support**

It would be nice to be able to make use of Gmail's COMPRESS (RFC4978) functionality.  I made a quick assessment of the `JavaMail` source that is responsible for this (Protocol.java I think) and changing it to support compression might be a bit beyond my talents right now.  I also got the impression it might not be worth the effort:

[Is it worthwhile using IMAP COMPRESS (DEFLATE)](http://stackoverflow.com/q/7386268/292219)?.

  * **Produce an efficient thread gathering mechanism**

I have a [working method](http://code.google.com/p/java-gmail-imap/wiki/DisplayingGmailThreadBasedView) to gather the top X threads from Gmail.  It would be really good if I could improve on this method.  Google Apps Script has a method to retrieve a range of Inbox threads ([getInboxThreads()](http://code.google.com/googleapps/appsscript/class_gmailapp.html#getMessagesForThread#getInboxThreads)) which leads me to suspect that there are features in their IMAP implementation that Google has not made public.

I'd be very grateful for any suggestions: [Searching for optimal method for X most recently active Gmail threads](http://stackoverflow.com/q/7322001/292219)


---


Enjoy!

