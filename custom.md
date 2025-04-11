# **คู่มือฉบับสมบูรณ์: การปรับแต่งและคอมไพล์ XNU Kernel บน macOS (Apple Silicon - ARM64)**
## **I. บทนำสู่การคอมไพล์ XNU บน Apple Silicon**
### **A. XNU Kernel: หัวใจของระบบปฏิบัติการ Apple**
XNU คือเคอร์เนล (Kernel) แบบไฮบริดที่เป็นแกนกลางของระบบปฏิบัติการตระกูล Apple ทั้ง macOS, iOS, iPadOS, watchOS, tvOS และ visionOS <sup>1</sup> ชื่อ XNU ย่อมาจาก "X is Not Unix" ซึ่งเป็นชื่อดั้งเดิมตั้งแต่สมัย NeXTSTEP ก่อนที่ macOS จะได้รับการรับรองมาตรฐาน UNIX อย่างเป็นทางการ <sup>2</sup> เคอร์เนลนี้เป็นการผสมผสานระหว่าง Mach microkernel ที่พัฒนาโดย Carnegie Mellon University เข้ากับส่วนประกอบจากเคอร์เนล Berkeley Software Distribution (BSD) และ Application Programming Interface (API) แบบ Object-Oriented ที่เขียนด้วย C++ สำหรับการพัฒนาไดรเวอร์ ที่เรียกว่า IOKit <sup>2</sup> การออกแบบแบบไฮบริดนี้พยายามนำข้อดีของทั้งสถาปัตยกรรมแบบ Monolithic และ Microkernel มาใช้ เช่น ความสามารถในการส่งข้อความ (message passing) ของ Microkernel ที่ช่วยเพิ่มความเป็นโมดูลาร์และขยายขอบเขตการป้องกันหน่วยความจำ (memory protection) ไปยังส่วนต่างๆ ของ OS ได้มากขึ้น ขณะเดียวกันก็ยังคงรักษาความเร็วในการทำงานของ Monolithic kernel ไว้ได้ <sup>2</sup> Apple ได้เปิดเผยซอร์สโค้ดของ XNU เป็นส่วนหนึ่งของโครงการ Darwin ซึ่งเป็นระบบปฏิบัติการโอเพนซอร์สที่เป็นรากฐานของ OS ต่างๆ ของ Apple <sup>5</sup>
### **B. ทำไมต้องคอมไพล์ XNU?**
การคอมไพล์ XNU Kernel จากซอร์สโค้ดเปิดโอกาสให้นักพัฒนา นักวิจัยความปลอดภัย และผู้ที่สนใจในระดับลึก สามารถ:

- **ทำความเข้าใจการทำงานระดับลึก:** การศึกษาและคอมไพล์ซอร์สโค้ดช่วยให้เข้าใจกลไกการทำงานพื้นฐานของ macOS และระบบปฏิบัติการอื่นๆ ของ Apple ได้อย่างถ่องแท้ <sup>6</sup>
- **การดีบักระดับเคอร์เนล:** ช่วยในการค้นหาและแก้ไขปัญหาในไดรเวอร์หรือส่วนประกอบอื่นๆ ที่ทำงานใน Kernel space ซึ่งเป็นเรื่องยากหากไม่มีซอร์สโค้ดและการดีบักที่เหมาะสม <sup>7</sup>
- **การวิจัยด้านความปลอดภัย:** การเข้าถึงซอร์สโค้ดช่วยให้นักวิจัยสามารถวิเคราะห์หาช่องโหว่ ทดสอบเทคนิคการโจมตี หรือพัฒนากลไกป้องกันใหม่ๆ ได้ <sup>9</sup>
- **การวิเคราะห์ประสิทธิภาพ:** สามารถปรับแต่งเคอร์เนลเพื่อวัดผลหรือปรับปรุงประสิทธิภาพในด้านต่างๆ ได้ <sup>4</sup>
- **การมีส่วนร่วมในโครงการโอเพนซอร์ส:** ผู้ที่สนใจสามารถพัฒนา แก้ไข หรือปรับปรุง XNU และส่งการเปลี่ยนแปลงกลับไปยังชุมชน Darwin/XNU ได้ <sup>10</sup>
### **C. ความแตกต่างของการคอมไพล์บน Apple Silicon (ARM64)**
การคอมไพล์ XNU บนสถาปัตยกรรม Apple Silicon (ARM64) มีความแตกต่างและซับซ้อนกว่าบนสถาปัตยกรรม Intel (x86\_64) อย่างมีนัยสำคัญ ประเด็นสำคัญที่ต้องพิจารณาคือ:

- **Kernel Debug Kit (KDK):** การคอมไพล์สำหรับ Apple Silicon *จำเป็นต้องใช้* KDK ซึ่งเป็นชุดเครื่องมือและส่วนประกอบเพิ่มเติมที่ไม่ได้มาพร้อมกับ Xcode ปกติ KDK อาจมี header files, ไลบรารี หรือแม้กระทั่งส่วนประกอบที่เป็น closed-source ที่จำเป็นสำหรับการลิงก์และสร้างเคอร์เนลให้ทำงานบนฮาร์ดแวร์เฉพาะได้ <sup>11</sup>
- **เครื่องมือ kmutil:** การจัดการ Kernelcache (KC) บน Apple Silicon ใช้เครื่องมือ kmutil เป็นหลัก ซึ่งมาแทนที่เครื่องมือเดิมอย่าง kextcache ที่ใช้บน Intel <sup>11</sup> การสร้างและติดตั้ง KC ที่กำหนดเองต้องทำผ่าน kmutil
- **Secure Boot Architecture:** สถาปัตยกรรม Secure Boot ของ Apple Silicon มีความเข้มงวดและแตกต่างจาก Intel การทดสอบเคอร์เนลที่คอมไพล์เองจำเป็นต้องปรับลดระดับความปลอดภัยของระบบผ่าน Recovery Mode และใช้คำสั่ง kmutil configure-boot แทนที่จะใช้ bless --nextonly ใน user space เหมือนในอดีต <sup>11</sup>
- **Platform Identifiers:** การสร้างเคอร์เนลสำหรับ Apple Silicon มักจะต้องระบุ Platform Identifier ที่เฉพาะเจาะจงสำหรับชิป M-series หรือ Mac รุ่นนั้นๆ <sup>11</sup>

การเปลี่ยนแปลงเหล่านี้สะท้อนให้เห็นว่า Apple ได้ใช้ประโยชน์จากการเปลี่ยนผ่านสู่สถาปัตยกรรมของตนเองเพื่อเพิ่มการควบคุมกระบวนการบูตและเคอร์เนลให้รัดกุมยิ่งขึ้น มีการผสานรวมฮาร์ดแวร์และซอฟต์แวร์ที่ลึกซึ้งกว่าเดิม และอาจปิดบังรายละเอียดการทำงานบางส่วน ทำให้กระบวนการสำหรับนักพัฒนาภายนอกมีความท้าทายมากขึ้นเมื่อเทียบกับยุค Intel <sup>12</sup>
### **D. ขอบเขตและคำเตือน**
คู่มือนี้มุ่งเน้น *เฉพาะ* การคอมไพล์และทดสอบ XNU Kernel บนเครื่อง Mac ที่ใช้ Apple Silicon (ARM64) การดำเนินการตามคู่มือนี้มีความเสี่ยงสูงและต้องอาศัยความเชี่ยวชาญทางเทคนิคอย่างมาก ผู้ใช้ต้องยอมรับความเสี่ยงที่อาจเกิดขึ้นได้ เช่น:

- **ระบบไม่เสถียร:** เคอร์เนลที่ปรับแต่งเองอาจทำงานผิดพลาด ทำให้ระบบค้าง หรือรีบูตเอง
- **ข้อมูลสูญหาย:** ความผิดพลาดในกระบวนการอาจนำไปสู่ความเสียหายของข้อมูล
- **เครื่องใช้งานไม่ได้ (Bricking):** แม้ว่าโดยทั่วไปจะสามารถกู้คืนผ่าน DFU Mode ได้ แต่ก็มีความเสี่ยงที่เครื่องจะบูตไม่ขึ้นอย่างถาวรในกรณีที่ร้ายแรง
- **การรับประกันสิ้นสุด:** การปรับเปลี่ยนซอฟต์แวร์ระบบในระดับนี้อาจทำให้การรับประกันจาก Apple สิ้นสุดลง <sup>17</sup>

**ขอแนะนำอย่างยิ่งให้ดำเนินการบนเครื่องที่ไม่ใช่เครื่องหลักสำหรับการทำงาน หรือทดสอบใน Virtual Machine หากเป็นไปได้** (แม้ว่า VM จะมีข้อจำกัดในการทดสอบเคอร์เนลบางอย่างก็ตาม <sup>18</sup>) ผู้ใช้ต้องมีความเข้าใจอย่างลึกซึ้งเกี่ยวกับระบบปฏิบัติการ การเขียนโปรแกรมระดับต่ำ และยอมรับความเสี่ยงทั้งหมดด้วยตนเอง
## **II. สิ่งจำเป็นเบื้องต้นและการตั้งค่า**
### **A. ฮาร์ดแวร์และซอฟต์แวร์**
- **เครื่อง Mac:** ต้องเป็นเครื่อง Mac ที่ใช้ชิป Apple Silicon (ตระกูล M1, M2, M3, M4) <sup>16</sup>
- **macOS:** ติดตั้ง macOS เวอร์ชันล่าสุดที่เข้ากันได้กับ XNU source และ KDK ที่ต้องการ (เช่น Ventura, Sonoma) ความเข้ากันได้ของเวอร์ชันเป็นสิ่งสำคัญอย่างยิ่ง (ดูหัวข้อ IX.B)
- **พื้นที่ดิสก์:** ต้องมีพื้นที่ว่างเพียงพอสำหรับซอร์สโค้ด, build artifacts, Xcode, SDKs, และ KDKs ซึ่งอาจใช้พื้นที่หลายสิบ GB **แนะนำให้สร้าง APFS Volume แบบ Case-sensitive** สำหรับเก็บซอร์สโค้ด XNU เพื่อหลีกเลี่ยงปัญหาที่อาจเกิดขึ้นระหว่างการคอมไพล์ <sup>52</sup>
### **B. Xcode และ Command Line Tools**
- **ติดตั้ง Xcode:** ดาวน์โหลดและติดตั้ง Xcode เวอร์ชันล่าสุดที่เกี่ยวข้องกับ macOS และ XNU ที่ต้องการ จาก App Store หรือเว็บไซต์ Apple Developer <sup>11</sup> การจับคู่เวอร์ชันที่ถูกต้องเป็นสิ่งสำคัญ (ดูหัวข้อ IX.B)
- **Command Line Tools:** โดยปกติจะติดตั้งมาพร้อม Xcode แต่สามารถติดตั้งแยกได้ด้วยคำสั่ง xcode-select --install ใน Terminal <sup>17</sup>
- **เลือก Xcode ที่ใช้งาน:** กำหนด Xcode เวอร์ชันที่ต้องการใช้งานด้วยคำสั่ง:
  Bash
  sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
  (ปรับ path ตามตำแหน่งที่ติดตั้ง Xcode) <sup>11</sup>
- **ยอมรับ License:** ยอมรับข้อตกลงการใช้งาน Xcode ด้วยคำสั่ง:
  Bash
  sudo xcodebuild -license
  <sup>11</sup>
- **ตรวจสอบ SDK Path:** ตรวจสอบว่าระบบรู้จักตำแหน่งของ macOS SDK อย่างถูกต้องด้วยคำสั่ง:
  Bash
  xcrun -sdk macosx --show-sdk-path
  <sup>11</sup>
### **C. Kernel Debug Kit (KDK)**
- **ความจำเป็น:** การคอมไพล์ XNU สำหรับ Apple Silicon *จำเป็นต้องใช้ KDK* <sup>11</sup> KDK เป็นชุดเครื่องมือพิเศษที่มี header files, ไลบรารี หรือส่วนประกอบ closed-source บางอย่างที่จำเป็นสำหรับการสร้างเคอร์เนลให้เข้ากับฮาร์ดแวร์ Apple Silicon รุ่นต่างๆ <sup>12</sup>
- **การจับคู่เวอร์ชัน:** **สำคัญมาก:** KDK ที่ใช้ *ต้องตรงกับเวอร์ชันและ Build Number ของ macOS* ที่กำลังใช้งานและต้องการคอมไพล์ XNU ให้รองรับ สามารถตรวจสอบเวอร์ชันปัจจุบันได้ด้วยคำสั่ง sw\_vers <sup>7</sup> การใช้ KDK ที่ไม่ตรงกันจะทำให้คอมไพล์ไม่ผ่าน หรือเกิด Kernel Panic ตอนบูต <sup>15</sup>
- **ดาวน์โหลด:** ดาวน์โหลดไฟล์ .pkg ของ KDK เวอร์ชันที่ถูกต้องได้จากหน้า Apple Developer Downloads (ค้นหา "Kernel Debug Kit") จำเป็นต้องใช้ Apple ID (อาจใช้ Free Tier ได้) <sup>7</sup>
- **ติดตั้ง:** ติดตั้ง KDK package โดยปกติจะติดตั้งไปที่ /Library/Developer/KDKs/KDK\_<OS\_Version>\_<Build>.kdk (เช่น /Library/Developer/KDKs/KDK\_14.4\_23E214.kdk) <sup>7</sup> จดจำ path นี้ไว้สำหรับตั้งค่า Environment Variable KDKROOT
### **D. การจัดการ Dependencies อื่นๆ (บริบททางประวัติศาสตร์และการตรวจสอบ)**
- **บริบทจากอดีต:** คู่มือเก่าๆ (โดยเฉพาะสำหรับ Intel หรือ macOS เวอร์ชันก่อนหน้า) มักระบุว่าจำเป็นต้องดาวน์โหลดและคอมไพล์เครื่องมือจาก dtrace (เช่น ctfconvert, ctfdump, ctfmerge), AvailabilityVersions, และ libfirehose จาก libdispatch ด้วยตนเองก่อน <sup>5</sup> ตัวอย่างขั้นตอนการคอมไพล์เหล่านี้สามารถดูได้จาก <sup>17</sup>
- **สถานการณ์ปัจจุบัน (Apple Silicon):** สำหรับ macOS เวอร์ชันใหม่ๆ บน Apple Silicon นั้น Dependencies เหล่านี้ *มีแนวโน้มสูงว่าจะถูกรวมอยู่ใน KDK* หรือถูกจัดการโดย build system ของ XNU ไปแล้ว เนื่องจากไม่มีการกล่าวถึงอย่างชัดเจนว่าต้องคอมไพล์ด้วยตนเองใน README ของ XNU หรือในคู่มือสำหรับ Apple Silicon ล่าสุด <sup>11</sup>
- **คำแนะนำ:** **ให้ลองคอมไพล์ XNU โดยตรงก่อน โดยยังไม่ต้องคอมไพล์ Dependencies เก่าเหล่านี้** หากพบข้อผิดพลาดในการคอมไพล์ที่เกี่ยวข้องกับ CTF tools, AvailabilityVersions, หรือ firehose\_buffer\_private.h *จึงค่อยย้อนกลับไป* ทำตามขั้นตอนในคู่มือเก่า <sup>17</sup> โดยปรับ path ให้เข้ากับโครงสร้างของ Xcode/SDK ปัจจุบัน

การเปลี่ยนแปลงนี้ชี้ให้เห็นว่ากระบวนการ build สำหรับ Apple Silicon น่าจะถูกปรับปรุงให้คล่องตัวขึ้น หรือรวมส่วนประกอบที่จำเป็นไว้ใน KDK แล้ว ซึ่งช่วยลดขั้นตอนการตั้งค่าเบื้องต้นเมื่อเทียบกับกระบวนการบน Intel ในอดีต <sup>5</sup> อย่างไรก็ตาม การทราบถึง Dependencies ในอดีตยังคงมีประโยชน์สำหรับการแก้ไขปัญหา build ที่อาจเกิดขึ้น
## **III. การจัดหาซอร์สโค้ด XNU**
### **A. การระบุเวอร์ชัน XNU ที่ถูกต้อง**
จำเป็นต้องใช้ซอร์สโค้ด XNU เวอร์ชันที่ตรงกับ macOS และ KDK ที่ติดตั้งไว้ ใช้คำสั่ง uname -a ใน Terminal เพื่อดูเวอร์ชันของเคอร์เนลที่กำลังทำงานอยู่ ตัวอย่างเช่น:



Darwin MyMac.local 23.4.0 Darwin Kernel Version 23.4.0: Fri Mar 15 00:11:05 PDT 2024; root:xnu-10063.101.17~1/RELEASE\_ARM64\_T6020 arm64

ในตัวอย่างนี้ เวอร์ชันหลักของ XNU คือ 10063.101.17 ซึ่งเป็นเวอร์ชันเป้าหมายที่ต้องหาซอร์สโค้ดให้ตรงกัน <sup>5</sup>
### **B. แหล่งดาวน์โหลดอย่างเป็นทางการ**
- **Apple Open Source Releases Page:** เป็นแหล่งข้อมูลหลักในอดีต เข้าไปที่ opensource.apple.com/releases/ เลือกเวอร์ชัน macOS ที่ต้องการ แล้วค้นหา tarball (.tar.gz) ของ XNU <sup>5</sup>
- **Apple OSS Distributions GitHub:** ปัจจุบัน Apple โฮสต์โครงการโอเพนซอร์สจำนวนมากบน GitHub รวมถึง XNU ด้วย เข้าไปที่ github.com/apple-oss-distributions/xnu <sup>1</sup> **วิธีที่แนะนำคือใช้ Git** เพื่อ clone repository และ checkout tag ที่ตรงกับเวอร์ชันเป้าหมาย (เช่น git checkout xnu-10063.101.17) การใช้ Git ช่วยให้ง่ายต่อการอัปเดตและติดตามการเปลี่ยนแปลง <sup>35</sup>
### **C. การแตกไฟล์ซอร์สโค้ด**
- หากดาวน์โหลดเป็น tarball ให้แตกไฟล์ด้วยคำสั่ง เช่น tar zxf xnu-10063.101.17.tar.gz <sup>11</sup>
- หากใช้ Git ให้ clone repository
- นำซอร์สโค้ดที่ได้ไปวางไว้ในตำแหน่งที่เหมาะสม **(แนะนำบน APFS Volume แบบ Case-sensitive)**
## **IV. ภาพรวมโครงสร้างซอร์สโค้ด XNU และการปรับแต่ง**
### **A. สถาปัตยกรรมระดับสูง**
XNU เป็นเคอร์เนลแบบไฮบริด <sup>2</sup> ประกอบด้วยส่วนหลักๆ ดังนี้:

- **Mach:** Microkernel หลัก จัดการ Inter-Process Communication (IPC), การจัดตารางเวลา (Scheduling), หน่วยความจำเสมือน (Virtual Memory), Remote Procedure Calls (RPC) <sup>4</sup> โค้ดส่วนใหญ่อยู่ในไดเรกทอรี osfmk/
- **BSD:** จัดเตรียม POSIX APIs, โมเดลโปรเซสแบบ Unix, ระบบไฟล์, Networking Stack (ซิงค์กับ FreeBSD) <sup>2</sup> โค้ดส่วนใหญ่อยู่ในไดเรกทอรี bsd/
- **IOKit:** เฟรมเวิร์ก Object-Oriented C++ สำหรับพัฒนา Device Drivers <sup>2</sup> โค้ดส่วนใหญ่อยู่ในไดเรกทอรี iokit/
- **Libkern:** ไลบรารี C++ ที่สนับสนุน IOKit และไดรเวอร์ <sup>3</sup> อยู่ใน libkern/
- **Networking:** ส่วนจัดการเครือข่าย เช่น TCP/IP stack <sup>4</sup>
### **B. ไดเรกทอรีซอร์สโค้ดที่สำคัญ**
นอกเหนือจากไดเรกทอรีหลักข้างต้น ยังมีไดเรกทอรีสำคัญอื่นๆ <sup>3</sup>:

- libsa/: โค้ดสำหรับ Kernel Bootstrap (กระบวนการเริ่มต้นเคอร์เนล)
- libsyscall/: ไลบรารีสำหรับ System Call Interface ฝั่ง User space
- security/: Security Policy Framework (MACF)
- EXTERNAL\_HEADERS/: Header files จากโครงการภายนอก
- tools/: เครื่องมือสำหรับ build และ test
### **C. แนวทางการปรับแต่ง (Customization)**
- **การแก้ไขโค้ด:** การปรับแต่ง XNU ทำได้โดยการแก้ไขซอร์สโค้ดภาษา C, C++, Objective-C หรือ Assembly ภายในไดเรกทอรีต่างๆ ที่กล่าวมา
- **ไฟล์ Configuration:** การเพิ่มไฟล์หรือโมดูลใหม่เข้าไปในเคอร์เนล อาจต้องแก้ไขไฟล์ configuration ใน osfmk/conf/ (สำหรับส่วน Mach) หรือ bsd/conf/ (สำหรับส่วน BSD) <sup>5</sup> อย่างไรก็ตาม การแก้ไขซอร์สโค้ดโดยตรงเป็นวิธีที่พบบ่อย
- **ตัวอย่างแนวคิด:**
  - การปรับเปลี่ยน Logic ของ Scheduler อาจเกี่ยวข้องกับไฟล์ใน osfmk/kern/
  - การแก้ไขพฤติกรรมของ Network Stack อาจอยู่ใน bsd/netinet/
  - การเพิ่มคำสั่ง kprintf เพื่อติดตามการทำงาน (Tracing) สามารถทำได้ในฟังก์ชันที่สนใจ
- **เครื่องมือช่วย:** การใช้เครื่องมืออย่าง ctags หรือ cscope (ซึ่งสามารถสร้างได้จากภายใน XNU source directory ด้วยคำสั่ง make tags หรือ make cscope <sup>31</sup>) จะช่วยในการสำรวจและทำความเข้าใจ codebase ขนาดใหญ่ได้ง่ายขึ้น
- **คำเตือน:** คู่มือนี้ *ไม่ได้* สอนการเขียนโค้ดเคอร์เนลโดยละเอียด การพัฒนาเคอร์เนลมีความซับซ้อนสูง ต้องการความรู้ด้าน C และ Systems Programming อย่างลึกซึ้ง <sup>37</sup> การแก้ไขที่ไม่ถูกต้องอาจนำไปสู่ระบบที่ไม่เสถียรหรือช่องโหว่ด้านความปลอดภัยได้ง่าย <sup>9</sup>

การปรับแต่ง XNU ให้มีประสิทธิภาพนั้น ไม่เพียงแต่ต้องอาศัยทักษะการเขียนโค้ดเท่านั้น แต่ยังต้องมีความเข้าใจในสถาปัตยกรรมแบบชั้น (Layered) และแบบไฮบริดของ XNU ด้วย เนื่องจาก XNU ไม่ใช่ codebase แบบ Monolithic เดียว แต่เป็นการรวมกันของ Mach, BSD และ IOKit <sup>2</sup> แต่ละส่วนมีปรัชญาการออกแบบ API และหน้าที่รับผิดชอบที่แตกต่างกัน การเปลี่ยนแปลงในส่วนหนึ่งอาจส่งผลกระทบหรือต้องการการเปลี่ยนแปลงในส่วนอื่นด้วย เช่น การเพิ่ม System Call ใหม่ อาจต้องแก้ไขทั้งในส่วน BSD (เพื่อประกาศ interface) และส่วน Mach (เพื่อ implement การทำงานจริง) การทำความเข้าใจว่าฟังก์ชันที่ต้องการแก้ไขนั้นอยู่ภายใต้การควบคุมของ Layer ใด (Mach, BSD หรือ IOKit) จึงเป็นสิ่งสำคัญในการค้นหาไฟล์ต้นทางที่ถูกต้องและทำการแก้ไขได้อย่างเหมาะสม
## **V. การกำหนดค่าสภาพแวดล้อมสำหรับคอมไพล์ (Build Environment)**
### **A. Environment Variables ที่จำเป็น**
การคอมไพล์ XNU โดยใช้คำสั่ง make จำเป็นต้องตั้งค่า Environment Variables ที่ถูกต้อง โดยเฉพาะอย่างยิ่งสำหรับ Apple Silicon <sup>11</sup>:

- **SDKROOT:** ระบุ path ไปยัง macOS SDK ที่ต้องการใช้ โดยทั่วไปตั้งค่าเป็น macosx หรือ macosx.internal หรือระบุ path เต็มที่ได้จากคำสั่ง xcrun
  - ตัวอย่าง: make SDKROOT=macosx... <sup>11</sup>
- **KDKROOT:** ระบุ Absolute path ไปยังไดเรกทอรีของ Kernel Debug Kit ที่ติดตั้งไว้ **สำคัญมากสำหรับ Apple Silicon**
  - ตัวอย่าง: export KDKROOT=/Library/Developer/KDKs/KDK\_14.4\_23E214.kdk หรือ make KDKROOT=/path/to/KDK... <sup>11</sup>
- **TARGET\_CONFIGS:** **Variable หลักสำหรับการ build บน Apple Silicon** ใช้กำหนด Configuration(s), สถาปัตยกรรม (Architecture), และ Platform เป้าหมาย รูปแบบคือ "VARIANT ARCH PLATFORM" สามารถระบุหลาย configuration ได้โดยคั่นด้วยเว้นวรรคภายในเครื่องหมายคำพูด
  - ตัวอย่าง: make TARGET\_CONFIGS="RELEASE ARM64 T8101" หรือ make TARGET\_CONFIGS="DEVELOPMENT ARM64 T6020" <sup>11</sup>
- **ARCH\_CONFIGS / KERNEL\_CONFIGS:** Variable เหล่านี้มักพบในเอกสารเก่าหรือใช้สำหรับการ build บน Intel <sup>11</sup> แต่สำหรับ Apple Silicon ควรใช้ TARGET\_CONFIGS ซึ่งเฉพาะเจาะจงกว่า
  - ตัวอย่าง (Intel): make ARCH\_CONFIGS=X86\_64 KERNEL\_CONFIGS=RELEASE <sup>11</sup>
### **B. การระบุ Apple Silicon Platform Identifiers**
การ build สำหรับ Apple Silicon จำเป็นต้องระบุ Platform Identifier ที่ถูกต้องใน TARGET\_CONFIGS ซึ่งจะแตกต่างกันไปตามรุ่นของชิปและ Mac:

- **ตัวระบุที่ทราบ:**
  - T8020: Developer Transition Kit (ADP3,2) (ยุค M1) <sup>11</sup>
  - T8101: M1 (MacBookPro17,1, MacBookAir10,1, Macmini9,1) <sup>7</sup>
  - T600x (เช่น T6000, T6001): ชิป M1 Pro/Max/Ultra? <sup>26</sup>
  - T602x (เช่น T6020): ชิป M2 series <sup>15</sup>
  - VMAPPLE: ตัวระบุที่ใช้ภายใน Virtualization framework VMs <sup>38</sup>
  - (อาจมีตัวระบุอื่นๆ ที่สอดคล้องกับชื่อไฟล์ใน /usr/share/kpep/ เช่น a14 (M1), a15 (M2), as1 (M3), as3 (M3 Pro/Max), as4 (M4) <sup>20</sup>)
- **วิธีค้นหา:**
  - รันคำสั่ง uname -v บนเครื่องเป้าหมาย และดูส่วนท้ายของ string เช่น .../RELEASE\_ARM64\_T8101 หมายถึง Identifier คือ T8101 <sup>7</sup>
  - ตรวจสอบไฟล์ .plist ในไดเรกทอรี /usr/share/kpep/ ซึ่งอาจมีชื่อตรงกับ Platform Identifier <sup>20</sup>
### **C. การปรับแต่ง Version String ของเคอร์เนล**
สามารถกำหนด version string ที่แสดงผลเมื่อใช้คำสั่ง uname -a ได้เอง โดยการตั้งค่า Variable RC\_DARWIN\_KERNEL\_VERSION แทนที่จะใช้ค่าที่ดึงมาจาก System.kext/Info.plist ของ SDK/KDK <sup>31</sup>

- ตัวอย่าง: make RC\_DARWIN\_KERNEL\_VERSION=10063.101.17-custom-build...
### **D. การแสดงผล Build Output แบบสี**
สามารถเปิดใช้งานการแสดงผล log การ build แบบมีสีเพื่อให้อ่านง่ายขึ้น โดยการตั้งค่า Environment Variable XNU\_LOGCOLORS=y หรือส่ง LOGCOLORS=y เป็น argument ให้กับ make <sup>3</sup>

**ตารางที่ 1: Environment Variables สำคัญสำหรับการ Build XNU บน Apple Silicon**

|**Variable**|**วัตถุประสงค์**|**ตัวอย่างค่า**|**แหล่งอ้างอิง**|
| :- | :- | :- | :- |
|SDKROOT|ระบุ Path ไปยัง macOS SDK|macosx หรือ macosx.internal|<sup>11</sup>|
|KDKROOT|ระบุ Path ไปยัง Kernel Debug Kit ที่ติดตั้งไว้ (จำเป็นสำหรับ Apple Silicon)|/Library/Developer/KDKs/KDK\_14.4\_23E214.kdk|<sup>11</sup>|
|TARGET\_CONFIGS|กำหนด Configuration, Architecture, และ Platform (จำเป็นสำหรับ Apple Silicon)|"RELEASE ARM64 T8101" หรือ "DEVELOPMENT ARM64 T6020"|<sup>11</sup>|
|RC\_DARWIN\_KERNEL\_VERSION (Optional)|กำหนด Version String ของเคอร์เนลเอง|"10063.101.17-mybuild"|<sup>31</sup>|
|LOGCOLORS (Optional)|เปิดใช้งานการแสดงผล Build Output แบบสี|y|<sup>13</sup>|

**ตารางที่ 2: Apple Silicon Platform Identifiers ที่พบบ่อย**

|**Platform Identifier**|**ชิป / รุ่น Mac ที่เกี่ยวข้อง (โดยประมาณ)**|**แหล่งอ้างอิง**|
| :- | :- | :- |
|T8020|Developer Transition Kit (M1 era)|<sup>11</sup>|
|T8101|M1 (MacBook Air, Mac Mini, MacBook Pro 13")|<sup>11</sup>|
|T600x|M1 Pro / M1 Max / M1 Ultra|<sup>26</sup>|
|T602x|M2 / M2 Pro / M2 Max / M2 Ultra|<sup>15</sup>|
|VMAPPLE|Virtualization Framework VMs|<sup>38</sup>|
|A14 / T8103?|M1 (อ้างอิงจาก kpep)|<sup>20</sup>|
|A15 / T8112?|M2 (อ้างอิงจาก kpep)|<sup>20</sup>|
|AS1 / T8122?|M3 (อ้างอิงจาก kpep)|<sup>20</sup>|
|AS3|M3 Pro / M3 Max (อ้างอิงจาก kpep)|<sup>20</sup>|
|AS4|M4 (อ้างอิงจาก kpep)|<sup>20</sup>|

*หมายเหตุ: การจับคู่ Platform Identifier กับชื่อชิป/รุ่น Mac อาจไม่สมบูรณ์ 100% ควรตรวจสอบด้วย uname -v บนเครื่องเป้าหมายเสมอ*
## **VI. การคอมไพล์ XNU Kernel**
### **A. คำสั่ง make**
คำสั่งหลักที่ใช้ในการเริ่มกระบวนการคอมไพล์คือ make ซึ่งต้องรันจากภายในไดเรกทอรีของ XNU source code <sup>3</sup>
### **B. การระบุ Build Configurations**
กำหนดรูปแบบการคอมไพล์ผ่าน KERNEL\_CONFIGS (ซึ่งเป็นส่วนหนึ่งของ TARGET\_CONFIGS สำหรับ Apple Silicon) <sup>3</sup>:

- **DEVELOPMENT:** (ค่าเริ่มต้น) เหมาะสำหรับการพัฒนาและดีบัก มีการเปิดใช้งาน Assertions (ASSERT=1) และสัญลักษณ์ดีบัก อาจมีการปรับแต่ง (Optimization) น้อยกว่า <sup>3</sup>
- **RELEASE:** Build ที่ปรับแต่งประสิทธิภาพสูงสุด ปิดใช้งาน Assertions ใกล้เคียงกับเคอร์เนลที่ใช้งานจริง <sup>11</sup>
- **DEBUG:** มีการตรวจสอบและ log การทำงานที่ละเอียดมากขึ้น อาจทำงานช้าลง <sup>3</sup>
- **PROFILE:** Build ที่รองรับการทำ Profiling
- **การ Build หลาย Config พร้อมกัน:** สามารถระบุหลาย configuration ใน TARGET\_CONFIGS ได้ เช่น TARGET\_CONFIGS="DEVELOPMENT ARM64 T8101 RELEASE ARM64 T8101"
### **C. Build Targets**
นอกจากการ build ตาม configuration ที่ระบุ (default target) ยังมี target อื่นๆ ที่น่าสนใจ:

- installhdrs: ติดตั้ง Kernel Headers ลงใน SDK path ซึ่งอาจจำเป็นหากต้องคอมไพล์ dependencies อื่นๆ ด้วยตนเอง <sup>11</sup>
- install\_kernels: ติดตั้งผลลัพธ์จากการ build (kernel binaries, symbols) ไปยังไดเรกทอรีที่ระบุด้วย DSTROOT <sup>31</sup> ตัวอย่าง: make install\_kernels DSTROOT=/tmp/xnu-install...
- tags / TAGS / cscope: สร้างไฟล์ tag สำหรับช่วยในการนำทาง (navigate) ซอร์สโค้ดด้วย editor หรือเครื่องมืออื่นๆ <sup>31</sup>
- xnu\_tests: คอมไพล์ชุด test programs ฝั่ง user-level สำหรับทดสอบฟีเจอร์ต่างๆ ของเคอร์เนล <sup>3</sup>
### **D. ผลลัพธ์จากการคอมไพล์ (Build Outputs)**
ไฟล์สำคัญที่ได้จากการคอมไพล์จะอยู่ในไดเรกทอรี BUILD/obj/ หรือโครงสร้างที่คล้ายกัน <sup>3</sup>:

- **kernel.<variant>.<platform>:** (เช่น kernel.development.t8101) ไฟล์ Binary หลักของเคอร์เนลที่คอมไพล์แล้วสำหรับ Apple Silicon รุ่นนั้นๆ **นี่คือไฟล์ที่จะนำไปสร้าง Kernelcache**
- **kernel.<variant>.unstripped.<platform>:** ไฟล์ Binary เดียวกัน แต่ยังไม่ได้ตัด (strip) Debug Symbols ออก จำเป็นสำหรับการดีบักด้วย LLDB
- **kernel.<variant>.<platform>.dSYM:** ไฟล์ Debug Symbol Bundle ในรูปแบบ DWARF (ค่าเริ่มต้น) ใช้โดย LLDB เพื่อแสดงข้อมูล symbolic ขณะดีบัก <sup>7</sup>

**ข้อสังเกต:** รูปแบบชื่อไฟล์ผลลัพธ์สำหรับ Apple Silicon (kernel.<variant>.<platform>) มีการระบุ Platform เป้าหมายอย่างชัดเจน <sup>7</sup> ซึ่งต่างจาก build บน Intel ที่อาจมีชื่อทั่วไปกว่า เช่น kernel.development <sup>31</sup> การระบุ platform นี้สอดคล้องกับการที่ต้องกำหนด Platform Identifier ใน TARGET\_CONFIGS <sup>11</sup> และเป็นการย้ำว่า build ที่ได้นั้นมีความจำเพาะเจาะจงกับฮาร์ดแวร์เป้าหมาย ซึ่งสำคัญอย่างยิ่งในการเลือกไฟล์เคอร์เนลที่ถูกต้องไปสร้าง Kernelcache ต่อไป
## **VII. การสร้าง Kernelcache (KC) ที่สามารถบูตได้**
### **A. บทบาทของ Kernelcache**
macOS บน Apple Silicon (และ Intel รุ่นใหม่ๆ) ใช้ Kernelcache (KC) แทน Prelinked Kernel แบบเดิม <sup>13</sup> KC คือ image ที่รวมเอาไฟล์เคอร์เนลที่คอมไพล์แล้ว มาเชื่อมโยง (link) เข้ากับ Kernel Extensions (KEXTs) ที่จำเป็นสำหรับการบูตระบบ <sup>3</sup> KC มีหลายประเภท เช่น Boot KC (BKC) สำหรับบูตระบบ, System KC (SKC) สำหรับ kext ระบบอื่นๆ (ใช้บน Intel เท่านั้น), และ Auxiliary KC (AKC) สำหรับ kext จาก Third-party <sup>14</sup> ในการทดสอบเคอร์เนลที่เราคอมไพล์เอง เราจะสร้าง Boot KC แบบกำหนดเอง (Custom Boot KC)
### **B. การใช้ kmutil create**
kmutil คือเครื่องมือหลักสำหรับจัดการและสร้าง KC บน macOS ยุคใหม่ <sup>11</sup> คำสั่งย่อย kmutil create ใช้สำหรับรวมเคอร์เนล binary เข้ากับ kext ที่ระบุ เพื่อสร้างเป็นไฟล์ .kc หรือ .im4p ที่สามารถบูตได้ <sup>39</sup>
### **C. พารามิเตอร์สำคัญของ kmutil create**
สำหรับการสร้าง Custom Boot KC บน Apple Silicon พารามิเตอร์ที่สำคัญมีดังนี้ <sup>11</sup>:

- -a arm64e: ระบุสถาปัตยกรรมเป้าหมายเป็น Apple Silicon
- -z: บีบอัด (compress) Kernelcache ที่ได้
- -V <variant>: ระบุ variant ของ KC (เช่น development, release) ต้องตรงกับ variant ของเคอร์เนลที่คอมไพล์มา และตรงกับวัตถุประสงค์ในการทดสอบ
- -n boot: ระบุประเภทของ collection เป็น 'boot'
- -k /path/to/kernel.<variant>.<platform>: **สำคัญมาก:** ระบุ path ไปยังไฟล์เคอร์เนล binary ที่เราคอมไพล์เอง
- -B /path/to/output.kc: ระบุ path และชื่อไฟล์สำหรับ Kernelcache ที่จะสร้างขึ้น (นามสกุล .kc หรือ .im4p <sup>39</sup>) ชื่อไฟล์ไม่สำคัญเท่ากับตอนใช้ bless <sup>11</sup>
- -r /System/Library/Extensions, -r /System/Library/DriverExtensions: ระบุไดเรกทอรีที่มี kext ของระบบ เพื่อให้ kmutil ค้นหาและรวม kext ที่จำเป็น
- -b <bundle\_id>: ระบุ Bundle Identifier ของ kext ที่ต้องการ *รวม* เข้าไปใน KC อย่างชัดเจน
- -x: ระบุว่า argument -b ที่ตามมาคือ kext ที่ต้องการ *ไม่รวม* (exclude) **(ข้อควรระวัง:** การใช้ -x ร่วมกับผลลัพธ์จาก kmutil inspect ใน <sup>11</sup> อาจทำให้สับสน ควรตรวจสอบให้แน่ใจว่าต้องการ include หรือ exclude kext จาก list นั้น โดยทั่วไปเราต้องการ *include* kext พื้นฐานจากระบบ ดังนั้นการใช้ -b โดยตรงน่าจะถูกต้องกว่า)
- --img4-encode: (อาจจำเป็นสำหรับ VM หรือกรณีพิเศษ) เข้ารหัส KC เป็นรูปแบบ .im4p <sup>39</sup>
- -s none: (อาจเกี่ยวข้องกับการ signing หรือ symbol stripping) <sup>26</sup>
### **D. การรวม Kexts ที่จำเป็น**
เคอร์เนล binary อย่างเดียวไม่สามารถบูตได้ ต้องรวมเข้ากับ kext ที่จำเป็นของระบบด้วย เทคนิคที่นิยมใช้คือการดึงรายการ kext จาก Kernelcache *ที่กำลังทำงานอยู่* (ซึ่งเป็น Release KC ของ Apple) ด้วยคำสั่ง kmutil inspect ซ้อนกัน <sup>11</sup>

- **โครงสร้างคำสั่ง:**
  Bash
  $(kmutil inspect -V release --no-header | grep -v "SEPHiber" | awk '{print " -b "$1; }')
- **คำอธิบาย:**
  - kmutil inspect -V release --no-header: แสดงรายการ Bundle ID ของ kext ทั้งหมดใน Release KC ปัจจุบัน โดยไม่แสดง header <sup>11</sup>
  - grep -v "SEPHiber": กรอง kext ที่ไม่เข้ากันออก เช่น SEPHiber ที่ไม่สามารถ link กับ open-source kernel ได้ <sup>11</sup> (อาจต้องกรอง kext อื่นๆ เพิ่มเติมหากพบปัญหา)
  - awk '{print " -b "$1; }': จัดรูปแบบ Bundle ID แต่ละรายการให้อยู่ในรูปแบบ -b <bundle\_id> เพื่อส่งเป็น argument ให้กับ kmutil create สำหรับการ *include* kext นั้นๆ <sup>11</sup>
- **คำสั่ง kmutil create ฉบับเต็ม (ตัวอย่าง):**
  Bash
  kmutil create -a arm64e -z -V development -n boot \
  ` `-k BUILD/obj/kernel.development.t8101 \
  ` `-B BUILD/MyCustomKernel.kc \
  ` `-r /System/Library/Extensions \
  ` `-r /System/Library/DriverExtensions \
  ` `$(kmutil inspect -V release --no-header | grep -v "SEPHiber" | awk '{print " -b "$1; }')

  *(ปรับ path และ platform identifier ตามความเหมาะสม)*

กระบวนการนี้สะท้อนให้เห็นว่า แม้ XNU kernel จะเป็น open source <sup>31</sup> แต่การทำให้มันทำงานได้จริงบน macOS จำเป็นต้องพึ่งพา kext จำนวนมากที่เป็น closed-source <sup>4</sup> การคอมไพล์ XNU ได้เพียงแค่ kernel binary <sup>11</sup> ซึ่งต้องนำมา "ประกอบร่าง" กับ kext ที่จำเป็นจาก OS ปัจจุบันผ่าน kmutil create <sup>11</sup> การใช้ kmutil inspect เพื่อดึงรายการ kext จาก release KC <sup>11</sup> จึงเป็นขั้นตอนสำคัญที่ช่วยให้มั่นใจว่าเคอร์เนลที่เราสร้างขึ้นเองนั้น มีไดรเวอร์และส่วนประกอบพื้นฐานครบถ้วนเพียงพอที่จะบูตและทำงานร่วมกับฮาร์ดแวร์และบริการต่างๆ ของระบบได้

**ตารางที่ 3: พารามิเตอร์ kmutil create สำคัญสำหรับการสร้าง Custom Boot KC**

|**พารามิเตอร์**|**คำอธิบาย**|**ตัวอย่างการใช้งาน**|**แหล่งอ้างอิง**|
| :- | :- | :- | :- |
|-a|ระบุสถาปัตยกรรมเป้าหมาย|arm64e|<sup>11</sup>|
|-z|บีบอัด Kernelcache|(ใส่ flag นี้)|<sup>11</sup>|
|-V|ระบุ Variant (development/release)|development หรือ release|<sup>11</sup>|
|-n|ระบุประเภท Collection|boot|<sup>11</sup>|
|-k|**ระบุ Path ไปยัง Custom Kernel Binary**|BUILD/obj/kernel.development.t8101|<sup>11</sup>|
|-B|ระบุ Path และชื่อไฟล์ Output KC|BUILD/MyCustomKernel.kc|<sup>11</sup>|
|-r|ระบุไดเรกทอรีสำหรับค้นหา Kexts|/System/Library/Extensions|<sup>11</sup>|
|-b|ระบุ Bundle ID ของ Kext ที่ต้องการ *Include*|-b com.apple.driver.AppleUSBXHCI|<sup>11</sup>|
|-x|ระบุว่า -b ที่ตามมาคือ Kext ที่ต้องการ *Exclude* (ใช้งานด้วยความระมัดระวัง)|-x -b com.apple.driver.DontIncludeThis|<sup>11</sup>|
|--img4-encode (Optional)|เข้ารหัส Output เป็นรูปแบบ .im4p|(ใส่ flag นี้)|<sup>39</sup>|
## **VIII. การทดสอบ Custom Kernel อย่างปลอดภัย**
การทดสอบเคอร์เนลที่คอมไพล์เองบน Apple Silicon เป็นกระบวนการที่ละเอียดอ่อนและต้องดำเนินการอย่างระมัดระวัง โดยอาศัยการทำงานร่วมกันระหว่าง Recovery Mode และการตั้งค่า Boot Arguments
### **A. การปรับเปลี่ยน Security Policy (ทำครั้งเดียว)**
การบูตด้วยเคอร์เนลที่ไม่ได้ลงนามโดย Apple จำเป็นต้องลดระดับความปลอดภัยของระบบ ซึ่งต้องทำจาก **Recovery Mode (1TR)** เท่านั้น (กดปุ่ม Power ค้างไว้ตอนเปิดเครื่อง) <sup>11</sup>:

- **วิธีใช้ Startup Security Utility (GUI):**
  - ใน Recovery Mode, ไปที่ Utilities > Startup Security Utility
  - เลือก Volume macOS เป้าหมาย, คลิก "Security Policy..."
  - เลือก "Reduced Security"
  - **ติ๊กถูกที่ "Allow user management of kernel extensions from identified developers"** (แม้จะทดสอบเคอร์เนล ไม่ใช่แค่ kext แต่ขั้นตอนนี้จำเป็นในการเปิดใช้งานการปรับแต่งระดับนี้) <sup>41</sup>
  - ยืนยันด้วยรหัสผ่าน Admin
- **วิธีใช้ Terminal (ทางเลือก/เสริม):**
  - csrutil disable: อาจยังจำเป็นสำหรับบางกรณี เพื่อปิด System Integrity Protection (ทำใน Recovery Terminal) <sup>11</sup>
  - bputil -a: อนุญาตให้ใช้ Custom Boot Arguments (ทำใน Recovery Terminal) <sup>11</sup>

การเปลี่ยนแปลง Security Policy นี้จะมีผลกับ OS ที่ติดตั้งบน Volume นั้นๆ จนกว่าจะมีการเปลี่ยนแปลงกลับ
### **B. การติดตั้ง Custom Kernelcache**
ขั้นตอนนี้ **ต้องทำจาก Terminal ใน Recovery Mode (1TR)** หลังจากสร้าง KC เสร็จแล้ว <sup>11</sup>:

1. **Unlock Volume (ถ้าใช้ FileVault):** หากเปิดใช้งาน FileVault ต้องปลดล็อก Volume ก่อน ด้วยคำสั่ง diskutil apfs unlockVolume /dev/diskXsY (แทน diskXsY ด้วย Identifier ที่ถูกต้อง) หรือปลดล็อกผ่าน GUI ของ Startup Security Utility ก่อนเปิด Terminal <sup>26</sup>
1. **ใช้คำสั่ง kmutil configure-boot:**
   Bash
   kmutil configure-boot -v /Volumes/Macintosh\ HD -c /path/to/your/custom.kc
   1. แทนที่ Macintosh HD ด้วยชื่อ Volume ที่ถูกต้อง
   1. แทนที่ /path/to/your/custom.kc ด้วย path เต็มไปยังไฟล์ Kernelcache ที่สร้างไว้ในขั้นตอน VII <sup>11</sup>
1. **ความหมาย:** คำสั่งนี้จะลงทะเบียน Custom KC ที่ระบุ ให้ถูกใช้ในการบูตครั้งถัดไปของ Volume นั้นๆ โดยเป็นการแก้ไข Boot Policy ของระบบ <sup>16</sup>
### **C. Boot Arguments ที่จำเป็น**
Boot Arguments ใช้ปรับเปลี่ยนพฤติกรรมของเคอร์เนลขณะบูต ตั้งค่าได้จาก Terminal ใน macOS ที่ทำงานอยู่ (หลังจากรัน bputil -a ใน Recovery แล้ว) ด้วยคำสั่ง sudo nvram boot-args="..." <sup>3</sup> Arguments สำคัญสำหรับการทดสอบ Custom XNU บน Apple Silicon ได้แก่:

- -v: **Verbose Boot** แสดง log การบูตของเคอร์เนลบนหน้าจอ แทนที่จะแสดงโลโก้ Apple จำเป็นอย่างยิ่งสำหรับการวินิจฉัยปัญหาการบูต <sup>3</sup>
- debug=0x144 (หรือค่าอื่น): เปิดใช้งานฟีเจอร์ Kernel Debugging มักใช้เพื่อให้ระบบรอการเชื่อมต่อจาก Debugger เมื่อเกิด Panic <sup>3</sup>
- kdp\_match\_name=enX (เช่น en8): ระบุ Network Interface สำหรับ Kernel Debugging ผ่านเครือข่าย (ต้องใช้อะแดปเตอร์เฉพาะ เช่น Thunderbolt Ethernet) <sup>3</sup>
- wdt=-1: ปิด Watchdog Timer ป้องกันไม่ให้เครื่องรีบูตอัตโนมัติขณะดีบักหรือเมื่อระบบค้าง <sup>8</sup>
- wlan.skywalk.enable=0: **จำเป็น:** ปิดใช้งาน Skywalk framework สำหรับ Wi-Fi เนื่องจากไม่มีอยู่ใน Open-source XNU <sup>11</sup>
- dk=0: ปิดใช้งาน DriverKit อาจช่วยเลี่ยงปัญหา Panic ที่เกี่ยวกับ Skywalk ได้ <sup>11</sup>
- -unsafe\_kernel\_text: (สำหรับ Sonoma ขึ้นไป) อาจจำเป็นสำหรับ Development Kernel เพื่อเลี่ยง Panic ที่เกี่ยวกับ Memory Management <sup>26</sup>
- kcsuffix=<suffix> / kernelsuffix=<suffix>: Argument เหล่านี้เคยใช้กับ bless เพื่อเลือกเคอร์เนลตามชื่อไฟล์ <sup>13</sup> แต่ kmutil configure-boot ไม่ได้ทำงานแบบนั้นโดยตรง อย่างไรก็ตาม kcsuffix อาจยังใช้เลือก *Built-in* Collection อื่นๆ ได้ (ถ้ามี) <sup>15</sup> **สำหรับการเลือก Custom KC ที่เราสร้างเอง ให้ใช้ kmutil configure-boot เป็นหลัก**
### **D. รีบูตและทดสอบ**
หลังจากตั้งค่า KC ใน Recovery Mode และตั้งค่า Boot Arguments (ถ้าจำเป็น) แล้ว ให้รีบูตเครื่อง Mac ตามปกติ สังเกตข้อความ verbose boot (ถ้าใช้ -v) ตรวจสอบการทำงานของระบบ และใช้ uname -a เพื่อยืนยันว่ากำลังรันเคอร์เนลเวอร์ชันที่กำหนดเอง (ถ้าตั้งค่า RC\_DARWIN\_KERNEL\_VERSION ไว้)
### **E. การย้อนกลับไปใช้ Stock Kernel**
หาก Custom Kernel บูตไม่ขึ้น หรือระบบไม่เสถียร:

1. บูตเข้า **Recovery Mode** อีกครั้ง
1. ใช้คำสั่ง kmutil configure-boot --clear (ตรวจสอบคำสั่งที่แน่นอนอีกครั้ง) หรือรัน kmutil configure-boot โดยชี้ไปยัง Stock KC (ถ้าทราบ path)
1. **วิธีที่แน่นอนกว่า:** รีเซ็ต Boot Policy กลับเป็นค่าเริ่มต้นด้วย bputil ใน Recovery เช่น bputil --full-security หรือคำสั่งที่เหมาะสมเพื่อกลับไปใช้ Full Security <sup>15</sup> ซึ่งจะบังคับให้ระบบใช้ Kernel Collection ที่ลงนามโดย Apple เท่านั้น
1. อาจต้องลบ Boot Arguments ที่มีปัญหาออกด้วย sudo nvram -d boot-args ใน Recovery

กระบวนการทดสอบบน Apple Silicon นี้แตกต่างจากบน Intel อย่างชัดเจน บน Intel การทดสอบมักทำได้จาก User space โดยใช้ bless --nextonly เพื่อบูตเคอร์เนลที่ระบุเพียงครั้งเดียว <sup>13</sup> แต่บน Apple Silicon เนื่องจากสถาปัตยกรรม Secure Boot ที่เข้มงวด <sup>16</sup> การเลือกและติดตั้งเคอร์เนลต้องทำจาก Recovery Mode ที่เชื่อถือได้ ผ่าน kmutil configure-boot <sup>11</sup> ซึ่งจะไปแก้ไข Boot Policy <sup>45</sup> ในขณะที่การปรับแต่งพฤติกรรมขณะบูตผ่าน Boot Arguments ยังคงตั้งค่าจาก OS หลัก (หลังจากได้รับอนุญาตจาก Recovery ด้วย bputil -a <sup>11</sup>) กระบวนการจึงแยกส่วนกัน: การเลือกเคอร์เนลทำใน Recovery, การปรับพฤติกรรมทำใน OS หลัก ซึ่งสะท้อนถึงแนวทางที่เน้นความปลอดภัยมากขึ้น

**ตารางที่ 4: Boot Arguments สำคัญสำหรับการทดสอบ Custom XNU บน Apple Silicon**

|**Argument**|**วัตถุประสงค์**|**ตัวอย่างการใช้งาน/ค่าที่ใช้**|**แหล่งอ้างอิง**|
| :- | :- | :- | :- |
|-v|แสดง Verbose Boot Log|(ใส่ flag นี้)|<sup>3</sup>|
|debug=|เปิดใช้งาน Kernel Debugging (รอ Debugger ตอน Panic)|0x144|<sup>3</sup>|
|kdp\_match\_name=|ระบุ Network Interface สำหรับ KDP Debugging|en8 (ขึ้นอยู่กับเครื่อง)|<sup>3</sup>|
|wdt=|ปิด Watchdog Timer|-1|<sup>8</sup>|
|wlan.skywalk.enable=|**ปิด Skywalk สำหรับ Wi-Fi (จำเป็นสำหรับ Open-source XNU)**|0|<sup>11</sup>|
|dk=|ปิด DriverKit (อาจช่วยเลี่ยงปัญหา Skywalk)|0|<sup>11</sup>|
|-unsafe\_kernel\_text|(Sonoma+) อาจจำเป็นสำหรับ Development Kernel เพื่อเลี่ยง Memory Panic|(ใส่ flag นี้)|<sup>26</sup>|
## **IX. ความท้าทาย ข้อจำกัด และการแก้ไขปัญหาที่พบบ่อย**
การคอมไพล์และทดสอบ XNU บน Apple Silicon มาพร้อมกับความท้าทายและข้อจำกัดหลายประการ:
### **A. อุปสรรคด้านความปลอดภัย**
- **System Integrity Protection (SIP):** ต้องปิดใช้งาน (csrutil disable) หรือตั้งค่าเป็น Reduced Security เพื่อให้สามารถโหลดโค้ดที่ไม่ได้ลงนามโดย Apple ได้ <sup>11</sup>
- **Secure Boot:** ต้องตั้งค่า Security Policy เป็น Reduced หรือ Permissive Security ผ่าน Recovery Mode <sup>11</sup> การใช้ Full Security จะป้องกันการบูตด้วยเคอร์เนลที่ไม่ได้ลงนาม
- **Kext Signing/Notarization:** หากการปรับแต่งเกี่ยวข้องกับการเพิ่มหรือแก้ไข KEXTs ตัว KEXTs นั้นๆ จะต้องได้รับการลงนามด้วย Developer ID Certificate ที่เปิดใช้งานสำหรับ KEXT Development และอาจต้องผ่านกระบวนการ Notarization จึงจะโหลดได้ แม้จะใช้ Reduced Security ก็ตาม ซึ่งเพิ่มความซับซ้อนอย่างมาก <sup>41</sup>
### **B. ความเข้ากันได้ของเวอร์ชัน**
- **Xcode, macOS, XNU Source, KDK:** ส่วนประกอบเหล่านี้ต้องมีความเข้ากันได้ และบ่อยครั้งที่ต้องตรงกัน *เป๊ะ* ทั้งเวอร์ชันและ Build Number <sup>5</sup> การใช้เวอร์ชันที่ไม่ตรงกันจะนำไปสู่ข้อผิดพลาดในการคอมไพล์หรือ Kernel Panic
- **ตรวจสอบ KDK:** ควรตรวจสอบก่อนว่า Apple ได้ปล่อย KDK สำหรับ macOS Build Number ที่ต้องการใช้งานหรือไม่ เนื่องจากไม่ได้มี KDK สำหรับทุก Point Release หรือ Beta <sup>7</sup>
### **C. ข้อกำหนด KDK และ Licensing**
- **KDK เป็นสิ่งจำเป็น:** ย้ำอีกครั้งว่าการ build สำหรับ Apple Silicon ต้องใช้ KDK <sup>11</sup>
- **ผลกระทบด้าน License:** ส่วนประกอบที่ถูกย้ายจาก Open-source release (ซึ่งใช้ Apple Public Source License - APSL) ไปอยู่ใน KDK จะอยู่ภายใต้ข้อตกลง Developer Agreement ของ Apple แทน ซึ่งไม่ใช่ APSL <sup>12</sup> สิ่งนี้จำกัดสิทธิ์ในการเผยแพร่ต่อ และอาจเปลี่ยนแปลงสิทธิ์การใช้งานเมื่อเทียบกับส่วนประกอบที่เป็น Open-source แท้ๆ นอกจากนี้ยังอาจเป็นวิธีที่ Apple ใช้เพื่อปิดบังส่วนขยาย ISA ที่เป็นกรรมสิทธิ์ หรือรายละเอียดการ implementation อื่นๆ <sup>12</sup>
### **D. ข้อจำกัดของ Open Source XNU**
ซอร์สโค้ด XNU ที่ Apple เปิดเผยนั้น *ไม่ได้มีฟีเจอร์ครบถ้วน* เหมือนกับเคอร์เนลที่ใช้งานจริง ทำให้เกิดข้อจำกัดในการทำงานเมื่อบูตด้วยเคอร์เนลที่คอมไพล์เอง <sup>11</sup>:

- **Rosetta 2:** ไม่รองรับการรันแอปพลิเคชัน x86\_64
- **Skywalk:** Framework เครือข่ายที่ใช้โดย Wi-Fi และไดรเวอร์อื่นๆ ไม่มีอยู่ ต้องใช้ Boot Argument wlan.skywalk.enable=0
- **Hibernate/Sleep:** ฟีเจอร์จัดการพลังงาน เช่น Hibernate อาจทำงานผิดปกติหรือไม่สมบูรณ์
- **SEP (Secure Enclave Processor):** การทำงานบางอย่างที่เกี่ยวข้องกับ SEP อาจไม่มีอยู่ (เช่น Kext SEPHiber ที่ถูกกรองออก <sup>11</sup>)
- **DriverKit (dk=0):** อาจต้องปิดใช้งานเนื่องจาก dependency กับส่วนที่ขาดหายไป เช่น Skywalk
- **ส่วนประกอบ Closed-source อื่นๆ:** มีความเป็นไปได้สูงว่ายังมี Framework หรือโค้ดสนับสนุนฮาร์ดแวร์อื่นๆ ที่ไม่ได้เปิดเผย
### **E. ข้อจำกัดในการดีบัก**
- การดีบักเคอร์เนลบน Apple Silicon มีข้อจำกัดมากกว่าบน Intel โดยเฉพาะในช่วงแรกๆ <sup>7</sup>
- การดีบักแบบ Interactive เต็มรูปแบบ (ตั้ง Breakpoint, Step-through code) อาจทำได้ยาก หรือมีข้อจำกัดเมื่อเทียบกับ GDB/LLDB บน Intel <sup>7</sup> วิธีที่นิยมคือการวิเคราะห์หลังเกิด Panic โดยใช้ KDP (Kernel Debugging Protocol) ผ่าน Network/Serial หรือวิเคราะห์ Crash Dump ที่สร้างโดย kdumpd <sup>7</sup>
### **F. การแก้ไขข้อผิดพลาดในการคอมไพล์ (Build Errors)**
- **Missing Headers:** มักเกิดจากการตั้งค่า SDKROOT/KDKROOT ไม่ถูกต้อง หรือขาด Dependencies (เช่น TrustCache/API.h, os/firehose\_buffer\_private.h หาก KDK/Dependencies ไม่ครบ) <sup>17</sup>
- **Linker Errors:** อาจเกิดจาก Architecture ไม่ตรงกัน, Library ใน KDK/SDK ไม่เข้ากัน, หรือ Build Flags ไม่ถูกต้อง
- **Tool Errors:** ปัญหาเกี่ยวกับ ctfconvert, ctfdump, ctfmerge หากจำเป็นต้องใช้ Dependencies เก่าแต่ไม่ได้ build หรือติดตั้งอย่างถูกต้อง <sup>17</sup>
- **แนวทางแก้ไข:** ตรวจสอบ Build Log อย่างละเอียด (ใช้ LOGCOLORS=y ช่วย), ค้นหาข้อผิดพลาดใน Developer Forums หรือ Mailing Lists, ดูตัวอย่างจากโครงการที่พยายามทำ Automated Build <sup>48</sup>
### **G. การรับมือกับ Kernel Panics**
- **สังเกตอาการ:** ระบบค้าง, รีบูตเอง, แสดงหน้าจอ Panic Log หลังรีบูต <sup>49</sup>
- **ใช้ Verbose Boot (-v):** เพื่อดูข้อความ Panic ขณะเกิดเหตุ <sup>26</sup>
- **วิเคราะห์ Panic Logs:** ไฟล์ log จะถูกเก็บไว้ที่ /Library/Logs/DiagnosticReports/ (มองหาไฟล์ที่ขึ้นต้นด้วย kernel\_)
- **ใช้ KDP Debugger:** หากตั้งค่าไว้ Debugger จะหยุดการทำงานเมื่อเกิด Panic <sup>3</sup>
- **การวิเคราะห์เบื้องต้น:** ดู Backtrace, ฟังก์ชัน/Kext ที่ทำให้เกิด Panic, และ Panic Reason String <sup>26</sup>

โดยสรุป การคอมไพล์และบูต Custom XNU Kernel บน Apple Silicon นั้นเป็นไปได้ แต่ถูกจำกัดอย่างมากจากความไม่สมบูรณ์ของซอร์สโค้ดที่เปิดเผย และสภาพแวดล้อมด้านความปลอดภัยที่เข้มงวด <sup>7</sup> เคอร์เนลที่ได้จะขาดฟังก์ชันสำคัญ เช่น Rosetta และ Skywalk ทำให้ต้องใช้ Workaround และจำกัดประโยชน์ใช้สอย การบังคับใช้ KDK และข้อจำกัดด้าน License ยิ่งทำให้ห่างไกลจากระบบ "เปิด" อย่างแท้จริง กระบวนการทดสอบที่ต้องอาศัย Recovery Mode และการปรับลดระดับความปลอดภัยก็สร้างความยุ่งยากและอาจไม่เป็นที่ต้องการสำหรับผู้ใช้ทั่วไป ทั้งหมดนี้สร้างสภาพแวดล้อมที่ท้าทายสำหรับนักวิจัยและนักพัฒนา เมื่อเทียบกับระบบนิเวศเคอร์เนลที่เปิดกว้างกว่า หรือแม้แต่สภาพแวดล้อมบน Mac Intel ในอดีต
## **X. สรุป**
### **A. สรุปกระบวนการ**
การปรับแต่งและคอมไพล์ XNU Kernel บน Apple Silicon เป็นกระบวนการที่ซับซ้อน ประกอบด้วยขั้นตอนหลักๆ ดังนี้:

1. **การเตรียมการ:** ติดตั้ง Xcode และ Command Line Tools เวอร์ชันที่ถูกต้อง และที่สำคัญคือ Kernel Debug Kit (KDK) ที่ตรงกับเวอร์ชัน macOS
1. **การจัดหาซอร์สโค้ด:** ดาวน์โหลดซอร์สโค้ด XNU เวอร์ชันที่ตรงกันจาก Apple Open Source หรือ GitHub
1. **การกำหนดค่า:** ตั้งค่า Environment Variables ที่จำเป็น โดยเฉพาะ SDKROOT, KDKROOT, และ TARGET\_CONFIGS ที่ระบุ Platform Identifier
1. **การคอมไพล์:** ใช้คำสั่ง make พร้อมระบุ Configuration ที่ต้องการ (Development, Release)
1. **การสร้าง Kernelcache:** ใช้ kmutil create เพื่อรวมเคอร์เนลที่คอมไพล์แล้วเข้ากับ Kexts ที่จำเป็นจากระบบ
1. **การทดสอบ:** บูตเข้า Recovery Mode เพื่อลดระดับความปลอดภัย และใช้ kmutil configure-boot เพื่อติดตั้ง KC ที่สร้างขึ้น จากนั้นรีบูตเพื่อทดสอบ โดยอาจต้องใช้ Boot Arguments เพิ่มเติม
### **B. ประเด็นสำคัญสำหรับ Apple Silicon**
- **KDK คือหัวใจหลัก:** ไม่สามารถคอมไพล์ XNU สำหรับ Apple Silicon ได้หากไม่มี KDK ที่ถูกต้อง
- **kmutil configure-boot ใน Recovery:** เป็นขั้นตอนบังคับสำหรับการติดตั้งและทดสอบ Custom Kernelcache
- **Platform Identifiers:** ต้องระบุ Identifier ที่ถูกต้องสำหรับ Mac รุ่นนั้นๆ ใน TARGET\_CONFIGS
- **ข้อจำกัดของ Open Source:** เคอร์เนลที่คอมไพล์ได้จะขาดฟังก์ชันสำคัญหลายอย่าง (Rosetta, Skywalk)
### **C. ข้อควรระวัง**
การดำเนินการตามคู่มือนี้มีความเสี่ยงสูง ผู้ใช้ควร:

- **สำรองข้อมูลสำคัญ** อย่างสม่ำเสมอ
- **ใช้เครื่องที่ไม่ใช่เครื่องหลัก** ในการทดลอง หากเป็นไปได้
- **ทำความเข้าใจผลกระทบด้านความปลอดภัย** ของการปรับเปลี่ยน Boot Policy และการปิด SIP
- **ยอมรับความเสี่ยง** ที่อาจเกิดขึ้นกับระบบและข้อมูล
### **D. แหล่งเรียนรู้เพิ่มเติม**
สำหรับผู้ที่ต้องการศึกษาเพิ่มเติมเกี่ยวกับ XNU และการพัฒนาเคอร์เนลบน macOS:

- **Kernel Programming Guide (Apple):** แม้บางส่วนอาจเก่า แต่แนวคิดพื้นฐานยังคงใช้ได้ <sup>5</sup>
- **XNU Source Code:** อ่าน README.md และเอกสารในไดเรกทอรี doc/ ภายในซอร์สโค้ด <sup>31</sup>
- **Kernel Debugging Guides:** ศึกษาการตั้งค่าและใช้งาน KDP <sup>7</sup>
- **หนังสือ:** "macOS and iOS Internals" (ชื่อเดิม "Mac OS X Internals") โดย Amit Singh และ Jonathan Levin <sup>6</sup>
- **ชุมชนออนไลน์:** Apple Developer Forums (ส่วน Kernel), Darwin Mailing Lists, Discord Servers ที่เกี่ยวข้อง (เช่น Hack Different <sup>15</sup>)


Dotmini Software  | Tirawat Nantamas github.com/XevilA
