# XLIST example #

To use the XLIST command you would do something like this:

```
...
 IMAPFolder imf;
 String[] atts;
 Folder[] f = store.getDefaultFolder().xlist("*");
 for (Folder fd : f) {
      System.out.println(fd.getFullName());
      imf = (IMAPFolder) fd;
      atts = imf.getAttributes();
      for(String a: atts){
         System.out.println(a);
      }
      System.out.println();
 }
...
```

this might result in:

```

[Gmail]
\Noselect
\HasChildren

[Gmail]/Borradores
\HasChildren
\HasNoChildren
\Drafts

[Gmail]/Destacados
\HasNoChildren
\Starred

[Gmail]/Enviados
\HasNoChildren
\Sent

[Gmail]/Importante
\HasNoChildren
\Important

[Gmail]/Papelera
\HasChildren
\HasNoChildren
\Trash

[Gmail]/Spam
\HasNoChildren
\Spam

[Gmail]/Todos
\HasNoChildren
\AllMail

```

I do not know why Trash and Drafts folders have both `HasChildren` and `HasNoChildren` attributes!