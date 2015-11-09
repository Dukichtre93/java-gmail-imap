# Displaying Gmail Thread Based View #

Displaying a Gmail inbox in the Gmail thread based style was my primary motivator for starting this project.  On this page you will find my code for the method that I use to acquire a list of the top X most recently active Gmail threads.  I hope you find this helpful.

```
package tests;

import com.google.code.com.sun.mail.imap.IMAPFolder;
import com.google.code.com.sun.mail.imap.IMAPMessage;
import com.google.code.com.sun.mail.imap.IMAPSSLStore;
import com.google.code.javax.mail.FetchProfile;
import com.google.code.javax.mail.Folder;
import com.google.code.javax.mail.Message;
import com.google.code.javax.mail.MessagingException;
import com.google.code.javax.mail.search.GmailThreadIDTerm;
import com.google.code.javax.mail.search.OrTerm;
import com.google.code.javax.mail.search.SearchTerm;
import com.google.common.collect.LinkedHashMultimap;
import tests.Constants;
import tests.xoauth.XoauthAuthenticator;
import java.io.IOException;
import java.util.ArrayList;
import java.util.Collection;
import java.util.HashSet;
import java.util.Map;
import java.util.TreeMap;

public class TestThreadFetch {

    private static final String INBOX = "INBOX";
    private static final FetchProfile fp;
    private static final FetchProfile minimalFetchProfile;

    static {
        fp = new FetchProfile();
        fp.add(FetchProfile.Item.ENVELOPE);
        fp.add(FetchProfile.Item.FLAGS);
        fp.add(IMAPFolder.FetchProfileItem.X_GM_THRID);
        fp.add(IMAPFolder.FetchProfileItem.X_GM_LABELS);
        minimalFetchProfile = new FetchProfile();
        minimalFetchProfile.add(IMAPFolder.FetchProfileItem.X_GM_THRID);
    }

    public static void main(String[] args) throws Exception {
        String username = Constants.getTestUser();
        String email = XoauthAuthenticator.getEmail(username);
        IMAPSSLStore imapSslStore = XoauthAuthenticator.getIMAPStore(username);
        if (imapSslStore != null) {
            execute(imapSslStore, 40, email);
        }
    }

    public static void execute(IMAPSSLStore store, int threadsToShow, final String user) throws IOException, MessagingException {
        Map<Long, Collection<Message>> threads = null;
        IMAPFolder folder = null;
        try {
            folder = (IMAPFolder) store.getFolder(INBOX);
            if (folder != null) {
                threads = searchThreads(folder, threadsToShow);                
                TreeMap<Long, Collection<Message>> threads_sorted = new TreeMap(threads);                
                int mLen;
                ArrayList<Message> msgs;
                IMAPMessage youngest, oldest;
                for (Collection<Message> ms : threads_sorted.descendingMap().values()) {
                    msgs = new ArrayList<Message>(ms);
                    mLen = msgs.size();
                    youngest = (IMAPMessage) msgs.get(mLen - 1);
                    oldest = (IMAPMessage) msgs.get(0);
                    System.out.println("Thread size: " + mLen);
                    System.out.println("Subject: " + oldest.getSubject());
                    System.out.println("Date: " + youngest.getReceivedDate());
                    String[] labels = youngest.getGoogleMessageLabels();
                    if (labels != null) {
                        for (String label : labels) {
                            System.out.println("*" + label);
                        }
                    }
                    System.out.println("Google Thread Id:" + Long.toHexString(youngest.getGoogleMessageThreadId()));
                    System.out.println();
                }
            }
        } finally {
            folder.close(true);
            store.close();
        }
    }

    public static Map<Long, Collection<Message>> searchThreads(final IMAPFolder folder, final int threadsToShow) throws MessagingException {
        LinkedHashMultimap<Long, Message> threads = LinkedHashMultimap.create();
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

    private static LinkedHashMultimap<Long, Message> fetchThreads(final IMAPFolder folder, final int threadsToShow, final int first, final int last, final LinkedHashMultimap<Long, Message> threads) throws MessagingException {
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
        // Create an array SearchTerm containing this batch of previously unread Google threads
        SearchTerm[] searchTerms = new SearchTerm[thrd_ids.size()];
        int c = 0;
        for (long thrId : thrd_ids) {
            searchTerms[c] = new GmailThreadIDTerm(thrId + "");
            c++;
        }
        if (searchTerms.length > 0) {
            // Fetch unread threads
            Message[] thrd_msgs = folder.search(new OrTerm(searchTerms));
            folder.fetch(thrd_msgs, fp);
            // Group individual fetched messages into message threads map           
            for (Message msg : thrd_msgs) {
                threads.put(((IMAPMessage) msg).getGoogleMessageThreadId(), msg);
            }
        }
        return threads;
    }
}
```

If you know of a more optimal method then I would be **VERY** interested.  Please contact me directly or if you would prefer then post it as an answer to my Stack Overflow question on the subject (link below).  Thanks in advance.

**[Searching for optimal method for X most recently active Gmail threads](http://stackoverflow.com/questions/7322001/gmail-imap-searching-for-optimal-method-for-x-most-recently-active-gmail-thread)**