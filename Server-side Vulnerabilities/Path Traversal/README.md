# **What is Path Traversal?**
It is also known as directory traversal where a paratmeter is used as a guiding light to access a directory which we wasn't supposed to. So attacker is tricking a web application into accessing files in this case you are kinda giving instructions to the web application to follow.

## Let's see this with a real life example
Imagine you visit a huge library.

You ask the librarian:

**Can I have the book Harry Potter from the Fantasy shelf?**

The librarian goes to:
~~~
Library/Fantasy/HarryPotter.pdf
~~~
Everything is fine.

Now imagine the librarian doesn't check what you ask for.

Instead you say:

**Go back one room... then another... then another... now bring me the manager's confidential salary records.**

The request becomes:
~~~
../
../
../
Manager/SalaryRecords.pdf
~~~
Instead of giving you a fantasy book, the librarian unknowingly gives you confidential files.

That is exactly what a Path Traversal attack does.

And as a technical example with the image attribute and filname parameter it always works very well.

~~~
<img src="/loadImage?filename=218.png">
~~~

What is it doing? The loadimage URL take the filename parameter and returns the contents of the file but it follows a path like **/var/www/images/218.png** with the help of API it does this in back. Now we can also recreate it with: **../../../etc/passwd** and if the passwd file with the other users credentials exists it will lead us there skipping all other directories that we don't know.



Some different path traversals paramerters other than "filename" which was shown in the portswigger lab:-

~~~
file
filename
path
filepath
dir
directory
folder
resource
page
template
include
module
view
document
doc
download
attachment
image
img
avatar
icon
logo
banner
css
style
js
script
theme
lang
locale
config
settings
log
backup
report
export
source
src
media
asset
content
~~~
And developers can invent completely custom names like:
~~~
userFile
profilePic
resume
invoice
manual
ebook
thumbnail
cover
dataFile
xml
csv
pdf
~~~
Or even something that gives no hint at all:

```
GET /api/get?id=42
```

Internally, the application might do something like:

```
id=42  →  /var/data/files/42.json
```

Even though the parameter is called id, it still controls which file is read.

Now, this ../ means to step up one level in the directory structure the three consecutive ../../../ sequences step up to the filesystem root

## Labs related with this topic in "Server-side Vulnerabilities"

In the lab it says: -

![Lab Overview](Images/LabOverview.png)

So we have to click on a image see the request and then change it in the middle so it will lead to the /passwd directory.
