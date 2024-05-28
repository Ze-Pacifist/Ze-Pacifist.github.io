+++
title = 'Rerecaptcha - Utctf 2022'
date = 2022-03-15 02:15:16
draft = false
tags = ["ocr","notes"]
+++

**tl;dr**

+ solve 1000 captchas to get flag.

<!--more-->

## source analysis

gen_challenge() used for generating captcha string.

gen_img() used for generating captcha image.


[http://web1.utctf.live:7132/](http://web1.utctf.live:7132/)

solve 1000 captchas to get flag.

![Untitled](rerecaptcha%20ae598444fed04520a50c56884c2bb492/Untitled.png)

![Untitled](rerecaptcha%20ae598444fed04520a50c56884c2bb492/Untitled%201.png)

cookie corresponds to captcha

cookie changes when captcha changes

so that means we can use cookie as sort of progress tracker/checkpoint to which we can come back to if captcha is wrong.

simply using image processing libraries wont work cuz its very unreadable.

to make it more readable, we can do some processing on the image.

[https://tesseract-ocr.github.io/tessdoc/ImproveQuality.html](https://tesseract-ocr.github.io/tessdoc/ImproveQuality.html)

noise removal, make it black and white to improve OCR(Optical Character Recognition).

notice that all images have same background

This means that if we have the original background image, we can take the difference / xor of our captcha to get a clean image.

script to collect images.

```python
import requests as re
import base64

url="http://web1.utctf.live:7132/"

for i in range(100):
	r=re.get(url)
	s=r.text.split('"data:image/png;base64, ')[1].split('"')[0]
	print(i)

	with open("bg_stuff/"+str(i)+".png","wb") as fh:
		fh.write(base64.b64decode(s))
```

ocr config options [https://stackoverflow.com/questions/44619077/pytesseract-ocr-multiple-config-options](https://stackoverflow.com/questions/44619077/pytesseract-ocr-multiple-config-options)

[https://wilsonmar.github.io/tesseract/](https://wilsonmar.github.io/tesseract/) oem 1 -neural engine mode.

extracting background.

```python
from PIL import Image
import pytesseract
from collections import Counter
pixel = [[{} for j in range(150)] for i in range(500)]
#key value pair for each pixel and finally find the maximum
#pixel[0][0][(0,1,2)]=1
#print(pixel[0][0])

for i in range(100):
	if (i%10 == 0): print(i)
	captchapath = "bg_stuff/"+str(i)+".png"
	trainingcaptcha = Image.open(captchapath)
	pixels = trainingcaptcha.load()
	for r in range(500):
		for c in range(150):
			p = pixels[r,c]
			if p in pixel[r][c]:
				pixel[r][c][p]+=1
			else:
				pixel[r][c][p]=1

inew=Image.new(mode="RGB", size=(500,150))
constructpixels = inew.load()

for r in range(500):
	for c in range(150):
		most = 0
		for p in pixel[r][c]:# Find the pixel which is repeating the most
			count = pixel[r][c][p]
			if count > most:
				most = count
				constructpixels[r,c] = p

inew.save("bg_stuff/background.png")
```

do some basic edits in photopea

[https://www.photopea.com/](https://www.photopea.com/)

especially where the 1st character is.

(show imagediff.py script and show the result and captcha and all)

make script to pack it up and send to server.

```python
import requests as re
from PIL import Image
import base64
from io import BytesIO
import pytesseract

url="http://web1.utctf.live:7132/"
url="http://172.20.0.2:5000/"

bestscore=0
cookie = {'session': ""}
prev_cookie={'session':""}

construct=Image.open("bg_stuff/background_edited.png")
constructpixels=construct.load()

while True:
	s=re.Session()

	r=s.get(url,cookies=cookie)

	pdata=r.text.split('"data:image/png;base64, ')[1].split('"')[0]
	captchaim=Image.open(BytesIO(base64.b64decode(pdata)))#reading base64decoded bytes as image
	captchaimpixels=captchaim.load()

	difference=Image.new(mode="RGB", size=(500,150))
	makediff=difference.load()

	for r in range(500):
		for c in range(150):
			cop = constructpixels[r,c]
			cap = captchaimpixels[r,c]
			if cop != cap:
				makediff[r,c] = (0,0,0)
			else:
				makediff[r,c] = (255,255,255)
					
	captchatext = pytesseract.image_to_string(difference, config='--oem 1 --psm 13')
	captchatext=captchatext[0:6]
	#print(captchatext)

	soln={'solution':captchatext}
	r=s.post(url,data=soln)
	score=int(r.text.split("You have solved ")[1].split(" ")[0])
	

	if score!=0:
		print(score,r.cookies['session'])
		prev_cookie=cookie
		cookie={'session':r.cookies['session']}
	else:
		print("fail",captchatext)
		cookie=prev_cookie
```