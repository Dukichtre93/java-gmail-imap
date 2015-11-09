# Investigating `JavaMail` 1.4.6 snapshot support for Gmail extensions

In you Maven pom.xml you need something like:

```
  <repositories>
    <repository>
      <id>Java.net snapshots</id>  
      <url>https://maven.java.net/content/repositories/snapshots/</url>
      <snapshots>
        <enabled>true</enabled>
      </snapshots>
    </repository>  
  </repositories>

  <dependencies>
    <dependency>
      <groupId>com.sun.mail</groupId>
      <artifactId>javax.mail</artifactId>
      <version>1.4.6-SNAPSHOT</version>
    </dependency>
    <dependency>
      <groupId>com.sun.mail</groupId>
      <artifactId>gimap</artifactId>
      <version>1.4.6-SNAPSHOT</version>
    </dependency>
  </dependencies>
```

My experience is that the new experimental Gmail support is quite easy to use.

Looks like labels behaviour has been updated as a result of my feedback.

  * [JAVAMAIL 1.4.6-SNAPSHOT, GMAIL GETLABELS PROBLEMS](http://kenai.com/projects/javamail/forums/forum/topics/532709-JavaMail-1-4-6-SNAPSHOT-Gmail-getLabels-problems).
  * http://kenai.com/projects/javamail/sources/mercurial/revision/480

```
package com.example;

import com.sun.mail.gimap.GmailFolder;
import com.sun.mail.gimap.GmailMessage;
import com.sun.mail.gimap.GmailSSLStore;
import java.util.Properties;
import javax.mail.FetchProfile;
import javax.mail.Folder;
import javax.mail.Message;
import javax.mail.MessagingException;
import javax.mail.Session;

public class App {

    public static void main(String[] args) throws MessagingException {
        Properties props = System.getProperties();
        props.setProperty("mail.store.protocol", "gimaps");
        //props.setProperty("mail.debug", "true");

        GmailSSLStore store = null;
        GmailFolder folder = null;
        try {
            Session session = Session.getDefaultInstance(props, null);
            store = (GmailSSLStore) session.getStore("gimaps");
            store.connect(Constants.getUsername(), Constants.getPassword());
            folder = (GmailFolder) store.getFolder("INBOX");
            folder.open(Folder.READ_ONLY);
            Message[] ms = folder.getMessages();
            FetchProfile fp = new FetchProfile();
            fp.add(GmailFolder.FetchProfileItem.MSGID);
            fp.add(GmailFolder.FetchProfileItem.THRID);
            fp.add(GmailFolder.FetchProfileItem.LABELS);

            folder.fetch(ms, fp);

            GmailMessage gm;
            String[] labels;

            for (Message m : ms) {
                gm = (GmailMessage) m;
                System.out.println(gm.getMsgId());

                // Hex version - useful for linking to Gmail
                //System.out.println(Long.toHexString(gm.getMsgId()));

                System.out.println(gm.getThrId());

                labels = gm.getLabels();
                if (labels != null) {
                    for (String label : labels) {
                        if (label != null) {
                            System.out.println("Label: " + label);
                        }
                    }
                }
            }
        } catch (MessagingException ex) {
        } finally {
            if (folder != null && folder.isOpen()) {
                folder.close(true);
            }
            if (store != null) {
                store.close();
            }
        }

    }
}
```