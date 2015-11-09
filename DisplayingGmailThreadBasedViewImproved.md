# Displaying Gmail Thread Based View - Improved #

I had a working method to [display a Gmail style thread based inbox](http://code.google.com/p/java-gmail-imap/wiki/DisplayingGmailThreadBasedView) but I was sure that that must be a way that the performance of it could be improved.  A helpful java-gmail-imap user made a very useful suggestion on how I could improve the efficiency of the downloads.

Usually in `JavaMail` messages obtained from a folder are light-weight empty references to the actual messages.  Any required attributes are populated "on-demand" and therefore require a further request/response round trip to the mail server.

You can pre-empt this second attribute fetching journey by using a `FetchProfile`.  By specifying a `FetchProfile` the attributes specified are pre-fetched in the results of your original fetch or term search.  This is good as it saves you a round trip to fetch these attributes later.  For example if you interested in the Google thread id's of your messages you can specify in your `FetchProfile` that the X\_GM\_THRID values should be returned.

In my case for my thread based view inbox I am only interested in the message's subject, Google thread id, labels, received date and from values.  So you would imagine that I can specify only these values in my `FetchProfile`?

The IMAP specification supports fine-grained attribute selection but `JavaMail`'s default behaviour does not.

In `JavaMail` if I require any message attribute that is part of the message envelope, a collection of message attributes including date, subject, from, sender, reply-to, to, cc, bcc, in-reply-to, and message-id then I need to pre-fetch all of the other envelope message attributes.

This doesn't sound a big deal but I am trying to improve the performance so it makes sense only to download the data I actually want.

I can achieve this by using IMAPFolder's doCommand and speaking directly to the IMAP server in the IMAP protocol.

A similar approach is described here:

[JavaMail IMAP over SSL quite slow - Bulk fetching multiple messages](http://stackoverflow.com/questions/8322836/javamail-imap-over-ssl-quite-slow-bulk-fetching-multiple-messages)

For demonstration purposes my example code contains two private classes (`CustomSearchProtocolCommand` and `GmailMessage`) that should probably be external classes.  It was necessary to create the `GmailMessage` POJO as an alternative to using `IMAPMessage` as a data transfer object.  This is because `IMAPMessage` is arranged in such a way as to only be populatable if `JavaMail` has downloaded the entire envelope (which is what I'm trying to avoid).

My artless example:

```

package tests;

import com.google.code.com.sun.mail.iap.ProtocolException;
import com.google.code.com.sun.mail.iap.Response;
import com.google.code.com.sun.mail.imap.IMAPFolder;
import com.google.code.com.sun.mail.imap.IMAPFolder.ProtocolCommand;
import com.google.code.com.sun.mail.imap.IMAPMessage;
import com.google.code.com.sun.mail.imap.IMAPSSLStore;
import com.google.code.com.sun.mail.imap.protocol.*;
import com.google.code.javax.mail.*;
import com.google.code.javax.mail.internet.MimeMessage;
import com.google.code.javax.mail.search.GmailThreadIDTerm;
import com.google.code.javax.mail.search.OrTerm;
import com.google.code.javax.mail.search.SearchTerm;
import com.google.common.collect.LinkedHashMultimap;
import tests.Constants;
import tests.xoauth.XoauthAuthenticator;
import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.util.*;
import java.util.logging.Level;
import java.util.logging.Logger;

public class TestThreadFetchCloseToMetal {

    private static final String INBOX = "INBOX";
    private static final FetchProfile minimalFetchProfile;

    static {
        minimalFetchProfile = new FetchProfile();
        minimalFetchProfile.add(IMAPFolder.FetchProfileItem.X_GM_THRID);
    }

    public static void main(String[] args) throws Exception {
        String username = Constants.getTestUser();
        String email = XoauthAuthenticator.getEmail(username);
        IMAPSSLStore imapSslStore = XoauthAuthenticator.getIMAPStore(username);
        if (imapSslStore != null) {
            execute(imapSslStore, 40);
        }
    }

    private static void execute(IMAPSSLStore store, int threadsToShow) throws IOException, MessagingException {
        Map<Long, Collection<GmailMessage>> threads;
        IMAPFolder folder = null;
        try {
            folder = (IMAPFolder) store.getFolder(INBOX);
            if (folder != null) {
                threads = searchThreads(folder, threadsToShow);
                TreeMap<Long, Collection<GmailMessage>> threads_sorted = new TreeMap(threads);
                int mLen;
                ArrayList<GmailMessage> msgs;
                GmailMessage youngest, oldest;
                for (Collection<GmailMessage> ms : threads_sorted.descendingMap().values()) {
                    msgs = new ArrayList<GmailMessage>(ms);
                    mLen = msgs.size();
                    System.out.println("Thread size: " + mLen);
                    youngest = msgs.get(mLen - 1);
                    oldest = msgs.get(0);
                    System.out.println("Subject: " + oldest.getSubject());
                    System.out.println("Date: " + youngest.getReceivedDate());
                    String[] labels = youngest.getLabels();
                    if (labels != null) {
                        for (String label : labels) {
                            System.out.println("*" + label);
                        }
                    }
                    System.out.println("Google Thread Id:" + Long.toHexString(youngest.getXGmThrid()));
                    System.out.println();
                }
            }
        } finally {
            folder.close(true);
            store.close();
        }
    }

    private static Map<Long, Collection<GmailMessage>> searchThreads(final IMAPFolder folder, final int threadsToShow) throws MessagingException {
        LinkedHashMultimap<Long, GmailMessage> threads = LinkedHashMultimap.create();
        int messagesToFetch = threadsToShow * 4;
        boolean stop = Boolean.FALSE;
        int msgCount, first, last;
        if (folder != null) {
            folder.open(Folder.READ_ONLY);
            msgCount = folder.getMessageCount();
            if (msgCount < messagesToFetch) {
                messagesToFetch = msgCount;
            }
            first = msgCount - messagesToFetch;
            if (first < 1) {
                first = 1;
            }
            last = msgCount;
            // Fetch messages in batches until desired thread count is reached (or INBOX is exhausted)          
            while (threads.size() < threadsToShow && !stop) {
                threads = fetchThreads(folder, threadsToShow, first + 1, last, threads);
                last = first - 1;
                first = last - messagesToFetch;
                if (first < 1) {
                    threads = fetchThreads(folder, threadsToShow, 1, last, threads);
                    stop = Boolean.TRUE;
                }
            }
        }
        return threads.asMap();
    }

    private static LinkedHashMultimap<Long, GmailMessage> fetchThreads(final IMAPFolder folder, final int threadsToShow, final int first, final int last, final LinkedHashMultimap<Long, GmailMessage> threads) throws MessagingException {
        // Fetch batch of messages
        Message[] msgs = folder.getMessages(first, last);
        HashSet<Long> thrd_ids = new HashSet<Long>();
        folder.fetch(msgs, minimalFetchProfile);
        int alreadyFetchedThreads = threads.size();
        IMAPMessage im;
        long googleThrdId;
        // Loop through fetched messages looking for message threads that have not already been read
        for (int i = msgs.length - 1; i > 0; i--) {
            im = (IMAPMessage) msgs[i];
            googleThrdId = im.getGoogleMessageThreadId();
            if (thrd_ids.size() < (threadsToShow - alreadyFetchedThreads)) {
                if (!threads.containsKey(googleThrdId)) {
                    thrd_ids.add(googleThrdId);
                }
            } else {
                break;
            }
        }
        // Create an array of SearchTerm containing this batch of previously unread Google threads
        SearchTerm[] searchTerms = new SearchTerm[thrd_ids.size()];
        int c = 0;
        for (long thrId : thrd_ids) {
            searchTerms[c] = new GmailThreadIDTerm(thrId + "");
            c++;
        }
        if (searchTerms.length > 0) {
            List<GmailMessage> thrd_msgs = (List<GmailMessage>) folder.doCommand(new CustomSearchProtocolCommand(new OrTerm(searchTerms)));
            for (GmailMessage gmailMessage : thrd_msgs) {
                threads.put(gmailMessage.getXGmThrid(), gmailMessage);
            }
        }
        return threads;
    }

    private static class CustomSearchProtocolCommand implements ProtocolCommand {

        private static final String FETCH_COMMAND = "INTERNALDATE X-GM-THRID X-GM-LABELS BODY.PEEK[HEADER.FIELDS (From Subject)]";
        private SearchTerm searchTerm;

        public CustomSearchProtocolCommand(SearchTerm searchTerm) {
            this.searchTerm = searchTerm;
        }

        public Object doCommand(IMAPProtocol protocol) throws ProtocolException {
            MessageSet[] msgSet;
            if (searchTerm != null) {
                try {
                    int[] sequences = protocol.search(searchTerm);
                    msgSet = MessageSet.createMessageSets(sequences);
                    ArrayList<GmailMessage> gmailMessages = new ArrayList<GmailMessage>();
                    Response[] r = protocol.fetch(msgSet, FETCH_COMMAND);
                    FetchResponse fetch;
                    BODY body;
                    GmailMessage gmailMessage;
                    ByteArrayInputStream is;
                    MimeMessage mm;
                    for (Response rr : r) {
                        if (rr instanceof FetchResponse) {
                            fetch = (FetchResponse) rr;
                            gmailMessage = new GmailMessage();
                            body = null;
                            for (int c = 0; c < fetch.getItemCount(); c++) {
                                if (fetch.getItem(c) instanceof INTERNALDATE) {
                                    gmailMessage.setReceivedDate(((INTERNALDATE) fetch.getItem(c)).getDate());
                                }
                                if (fetch.getItem(c) instanceof X_GM_THRID) {
                                    gmailMessage.setXGmThrid(((X_GM_THRID) fetch.getItem(c)).x_gm_thrid);
                                }
                                if (fetch.getItem(c) instanceof X_GM_LABELS) {
                                    gmailMessage.setLabels(((X_GM_LABELS) fetch.getItem(c)).x_gm_labels);
                                }
                                if (fetch.getItem(c) instanceof BODY) {
                                    body = (BODY) fetch.getItem(c);
                                }
                            }
                            is = body.getByteArrayInputStream();
                            mm = new MimeMessage(null, is);
                            gmailMessage.setFrom(mm.getFrom());
                            gmailMessage.setSubject(mm.getSubject());
                            gmailMessages.add(gmailMessage);
                        }
                    }
                    return gmailMessages;
                } catch (MessagingException ex) {
                    Logger.getLogger(TestThreadFetchCloseToMetal.class.getName()).log(Level.SEVERE, null, ex);
                }
            }
            return null;
        }
    }

    private static class GmailMessage {

        private String subject;
        private Address[] from;
        private Date receivedDate;
        private long xGmThrid;
        private String[] labels;

        public String getSubject() {
            return subject;
        }

        public void setSubject(String subject) {
            this.subject = subject;
        }

        public Address[] getFrom() {
            return from;
        }

        public void setFrom(Address[] from) {
            this.from = from;
        }

        public Date getReceivedDate() {
            return receivedDate;
        }

        public void setReceivedDate(Date receivedDate) {
            this.receivedDate = receivedDate;
        }

        public long getXGmThrid() {
            return xGmThrid;
        }

        public void setXGmThrid(long xGmThrid) {
            this.xGmThrid = xGmThrid;
        }

        public String[] getLabels() {
            return labels;
        }

        public void setLabels(String[] labels) {
            this.labels = labels;
        }
    }
}

```