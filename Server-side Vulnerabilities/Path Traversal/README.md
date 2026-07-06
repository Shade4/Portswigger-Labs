Some different path traversals paramerters other than "filename" which was shown in the portswigger lab:-


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

And developers can invent completely custom names like:

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

Or even something that gives no hint at all:

```
GET /api/get?id=42
```

Internally, the application might do something like:

```
id=42  →  /var/data/files/42.json
```

Even though the parameter is called id, it still controls which file is read.
