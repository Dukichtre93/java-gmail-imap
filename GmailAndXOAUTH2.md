Created: 6th November 2012

# How to achieve domain-wide delegation of authority on Gmail with OAuth 2.0 #

On my Google Apps for Education domain I have a requirement to present a user's Gmail inbox to the user in our enterprise portal.  The mechanism that supports this has recently changed, this page describes the new mechanism to access Gmail IMAP using OAuth 2.0.

On Google Apps for Business and Education domains it is currently possible to access the Gmail IMAP account of any domain user using two-legged OAuth 1.0a  (e.g. domain-wide delegation of authority using
[deprecated XOAUTH protocol](https://developers.google.com/google-apps/gmail/oauth_protocol) and [Google Mail Xoauth Tools](http://code.google.com/p/google-mail-xoauth-tools/)).  The method currently still works but it is being deprecated.

In order to bring Gmail in line with the other Google API services the Gmail servers have been updated to support OAuth 2.0, a new protocol XOAUTH2 has been introduced (See: [Adding OAuth 2.0 support for IMAP/SMTP and XMPP to enhance auth security](http://googledevelopers.blogspot.co.uk/2012/09/adding-oauth-20-support-for-imapsmtp.html), [XOAUTH2 Mechanism](https://developers.google.com/google-apps/gmail/xoauth2_protocol)).

Along with this new protocol is a new set of tools, [Google Mail OAuth2 Tools](http://code.google.com/p/google-mail-oauth2-tools/).

The following describes how to use XOAUTH2 and OAuth 2.0 to achieve the equivalent of 2-legged OAuth.

## Google APIs console ##

Firstly you need to visit the [Google API Console](https://code.google.com/apis/console/) and create a Client ID.  You will need to create the Client as a [Service account](https://code.google.com/p/google-api-java-client/wiki/OAuth2#Service_Accounts)

![http://java-gmail-imap.googlecode.com/svn/wiki/images/SelectApplicationType.png](http://java-gmail-imap.googlecode.com/svn/wiki/images/SelectApplicationType.png)

Make a note of the **Client ID**, **email address** and save the **private key** somewhere - you will need these later.

![http://java-gmail-imap.googlecode.com/svn/wiki/images/PublicPrivateKeys.png](http://java-gmail-imap.googlecode.com/svn/wiki/images/PublicPrivateKeys.png)

![http://java-gmail-imap.googlecode.com/svn/wiki/images/ServiceAccount.png](http://java-gmail-imap.googlecode.com/svn/wiki/images/ServiceAccount.png)

## Google Apps Dashboard : Manage third party OAuth Client access ##

Next the administrator of your Google for Business/Education domain needs to add your Client ID to the list of authorized API clients.

The "Client name" will be something like **484313896601.apps.gserviceaccount.com** and in order to access GMail via IMAP the scope will be `https://mail.google.com/`.  Once the administrator authorizes this Client you will have permission to **Email (Read/Write/Send)** as any user in that domain.

![http://java-gmail-imap.googlecode.com/svn/wiki/images/manage_api.gif](http://java-gmail-imap.googlecode.com/svn/wiki/images/manage_api.gif)

Now you can create the Java code.  Firstly you need to acquire the [Google Mail OAuth2 Tools](http://code.google.com/p/google-mail-oauth2-tools/) ([oauth2-java-sample-20120904.zip](http://google-mail-oauth2-tools.googlecode.com/files/oauth2-java-sample-20120904.zip)), I unpacked the source files from this (OAuth2Authenticator.java, OAuth2SaslClient.java and OAuth2SaslClientFactory.java) into my Maven project.  The only dependency of these classes is on `JavaMail` (it can easily be re-written to use this project's code).

Here is a snippet of my Maven pom.xml file containing the code dependencies.

```
<repositories>
    <repository>
        <id>googleapis</id>
        <url>http://mavenrepo.google-api-java-client.googlecode.com/hg/</url>
        <snapshots>
            <enabled>true</enabled>
        </snapshots>
    </repository>        
</repositories>  

<dependencies>
    <dependency>
        <groupId>javax.mail</groupId>
        <artifactId>mail</artifactId>
        <version>1.4.4</version>
    </dependency>
    <dependency>
        <groupId>com.google.api-client</groupId>
        <artifactId>google-api-client</artifactId>
        <version>1.12.0-beta</version>
    </dependency>
    <dependency>
        <groupId>com.google.http-client</groupId>
        <artifactId>google-http-client-jackson2</artifactId>
        <version>1.12.0-beta</version>
    </dependency>  
</dependencies>
```

`GoogleCredential` requires google-api-client.
`JacksonFactory` requires google-http-client-jackson2.

You should now be able to access the IMAP inbox of any Google domain user thus:

```
import com.google.api.client.googleapis.auth.oauth2.GoogleCredential;
import com.google.api.client.http.HttpTransport;
import com.google.api.client.http.javanet.NetHttpTransport;
import com.google.api.client.json.JsonFactory;
import com.google.api.client.json.jackson2.JacksonFactory;
import com.google.code.samples.oauth2.OAuth2Authenticator;
import com.sun.mail.imap.IMAPStore;
import java.io.File;
import java.io.IOException;
import java.security.GeneralSecurityException;

public class Test {

    private static final HttpTransport HTTP_TRANSPORT = new NetHttpTransport();
    private static final JsonFactory JSON_FACTORY = new JacksonFactory();
    private static final String GMAIL_SCOPE = "https://mail.google.com/";

    public static void main(String[] args) throws Exception {	
        String email = "test@electric-automotive.com";
        String authToken = getAccessToken(email);

        OAuth2Authenticator.initialize();

        IMAPStore imapSslStore = OAuth2Authenticator.connectToImap("imap.gmail.com",
                993,
                email,
                authToken,
                true);

        System.out.println("Successfully authenticated to IMAP.\n");
    }

    public static String getAccessToken(String email) throws GeneralSecurityException, IOException {
        GoogleCredential credential = new GoogleCredential.Builder().setTransport(HTTP_TRANSPORT)
                .setJsonFactory(JSON_FACTORY)
                .setServiceAccountId("484313896601@developer.gserviceaccount.com")
                .setServiceAccountScopes(GMAIL_SCOPE)
                .setServiceAccountPrivateKeyFromP12File(new File("privatekey.p12"))
                .setServiceAccountUser(email)
                .build();
        credential.refreshToken();
        return credential.getAccessToken();
    }
}
```