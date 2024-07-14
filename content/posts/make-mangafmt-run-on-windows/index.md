---
title: "Make Mangafmt Run on Windows"
date: 2024-07-14T18:26:42+07:00
draft: false
tags: ["mangafmt", "golang", "imagemagick"]
author: "teerapapc"
cover:
  image: "images/cover.png" # image path/url
  alt: "Cross-compilation build script" # alt text
  relative: true
tweetId: 
---

หลังจากเดือนก่อนปล่อย [mangafmt]({{< ref "project-mangafmt" >}}) ซึ่งพัฒนาบน [WSL2](https://en.wikipedia.org/wiki/Windows_Subsystem_for_Linux) และ Build/Run ได้บน Linux เท่านั้น ความตั้งใจถัดมาคือ อยากทำให้มันรันได้บน Windows/OSX ด้วย เพราะคนส่วนใหญ่ใช้ และส่วนตัวก็ใช้ Windows เป็นหลัก

หลังจากค้นๆ อยู่แป๊ปนึงก็พบว่า Go มัน cross-compile ง่ายมาก! ตามรูปปกด้านบนเลย แค่ set environment variable ชื่อ `GOOS` และ `GOARCH` แค่นั้นก็ build ได้ทั้งข้าม OS และ Architecture

แต่โลกไม่สวยงามแบบนั้น ถ้าโค้ดเรามีเรียกใช้ C library อยู่ ชีวิตจะยุ่งยากขึ้นมากในการ build และ lib ที่เราใช้คือ ImageMagick ซึ่งมันทั้งใหญ่และ dependencies ตามมาอีกเพียบ แถมมีเวอร์ชั่น 6 กับ 7 ซึ่งแต่ละ platform อาจจะใช้คนละเวอร์ชันกันอีก

หลังจากลองพยายามอยู่ซักพัก ก็ไม่เอาดีกว่า แก้เป็น pure Go ดีกว่า ข้อดีคือ build ง่าย และ executable ที่ออกมา คือรันได้เลย ไม่ต้องลง runtime dependencies เพราะงั้น **ต้องเอา ImageMagick ออก**

เราใช้ [ImageMagick](https://imagemagick.org/) ทำอยู่ 2 อย่างคือ image manipulation กับ image  rendering จาก PDF

## Image Manipulation with Pure Go

อันนี้ไม่ยาก เพราะ operation มันพื้นฐาน resize/crop/grayscale conversion/edge trimming/etc. หาได้จาก stdlib ของ Go เลย อันไหนไม่มี ก็เขียนเอง แกะๆจากโค้ด ImageMagick มาบ้างเช่นการเทียบสีแบบมี fuzz factor หรือการคำนวน distortion (RMSE)

สนุกดี ได้เรียนรู้เรื่อง [Premultiplied Alpha](https://nigeltao.github.io/blog/2022/premultiplied-alpha.html) หรือ [Dithering](https://en.wikipedia.org/wiki/Dither) เขียนแล้วก็ เทียบผลกับ ImageMagick ค่อยๆทำทีละ operation ไปจบครบ

{{< figure src="images/dither.png" caption="ตัวอย่าง Image Dithering" attr="CC BY-SA 4.0" attrlink="https://commons.wikimedia.org/w/index.php?curid=143581140" >}}

## PDF Format

สิ่งที่เราต้องการคือ ดึงแต่ละหน้าใน PDF ออกมาเป็นรูป แว๊บแรกคิดว่าไม่ยาก หา lib ซักตัวที่เป็น Go ก็น่าจะมี นั่งหาอยู่หลายวันก็พบว่า แม้ PDF จะเป็น open format (ISO32000) แต่ทุกวันนี้ในโลกมี lib ที่ใช้ read/render PDF อยู่ไม่กี่เจ้า (ไม่มี Go-based 😢) และไลเซนส์ก็เป็น [GPL](https://en.wikipedia.org/wiki/GNU_General_Public_License) หรือไม่ก็เสียตังค์

ImageMagick ก็ไม่ได้ทำตรงนี้เอง แต่เรียกคำสั่งให้ [Ghostscript](https://www.ghostscript.com/) ทำ
Google Chrome เองก็ใช้ของตัวเองชื่อ [PDFium](https://pdfium.googlesource.com/pdfium/) ซึ่งพัฒนาร่วมกับ [Foxit](https://en.wikipedia.org/wiki/Foxit_Software) (คนมีอายุหน่อยน่าจะเคยได้ยินชื่อ Foxit PDF)

หรือเราต้องเขียนเอง? นั่งเปิดอ่าน [PDF format spec](https://opensource.adobe.com/dc-acrobat-sdk-docs/standards/pdfstandards/pdf/PDF32000_2008.pdf) จริงจัง ก็พบว่า format มันคล้ายๆ HTML เลย คือ เป็น Tree ของ Object type ต่างๆ แต่เป็น Binary based ไม่ใช่ Text เหมือน HTML เราสามารถ parse tree แล้ว jump ไปตาม node ต่างๆได้ด้วย byte offset

แต่อ่านไปจนพอจะได้ไอเดีย ก็คิดว่าไม่เอาดีกว่า เพราะสิ่งที่ต้อง support และเข้าใจมันเยอะเกิน ทั้งเรื่อง Compression/ColorSpace/Font/Graphics/etc. ไม่ไหว นี่น่าจะอธิบายกว่าทำไม opensource library ที่มีส่วนใหญ่เป็นการ generate/append/edit ซึ่งเขียนง่ายกว่า ทำเท่าที่ต้องใช้ 

เลยเปลี่ยนแผนเป็นใช้ [os.exec](https://pkg.go.dev/os/exec) รันคำสั่งของ ImageMagick แทน ซึ่งข้อเสียคือ ก็ต้องลง ImageMagick เป็น runtime dependencies อยู่ดี แต่ข้อดีคือ ลงเฉพาะถ้า input เป็น PDF และ รองรับ tool ดัวอื่นได้ด้วย เพราะระหว่างที่ศึกษาเรื่องนี้ก็ไปเจอ [libvips](https://www.libvips.org/) ซึ่งเหมือนเป็น modern ImageMagick ทำอะไรได้คล้ายๆกัน แต่เร็วกว่า

## พอแค่นี้?

หลังจากเอา ImageMagick ออกและหาของมาทดแทนสำเร็จ ก็ทำให้ cross-compile ด้วยคำสั่งง่ายๆ คือ `go build` ชีวิต happy

พึ่งปล่อยเมื่อคืน [mangafmt v0.3.0](https://github.com/teerapap/mangafmt/releases/tag/v0.3.0)

มีอะไรอยากทำต่ออีกหน่อย ถ้ามีอะไรน่าสนใจ จะมาเล่าอีก 😄
