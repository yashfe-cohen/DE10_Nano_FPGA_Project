[🔙 חזרה לתוכן העניינים של הלינוקס](../README.md)

# בניית מערכת הקבצים (Debian RootFS)

יש הרבה סוגים של מערכות קבצים (RootFS) שאפשר לבנות עבור מערכות משובצות. במדריך הזה בחרתי להתמקד ב-**Debian**.
הבחירה הזו תיתן לנו סביבת עבודה מלאה ונוחה על ה-DE10-Nano, שמרגישה ממש כמו עבודה על מחשב לינוקס רגיל (בדומה ל-Raspberry Pi).

> 💡 **למה לא Buildroot או Yocto?**
> פלטפורמות כמו Buildroot ו-Yocto הן מעולות ונמצאות בשימוש נרחב בתעשייה, אבל הן מיועדות ליצור מערכות הפעלה סגורות, קטנות ומאוד ספציפיות (למשל לראוטר או ציוד תקשורת). הפרויקט שלנו דורש מערכת הפעלה כללית שנוח לפתח עליה ולהתקין בה חבילות באופן חופשי, ולכן Debian היא הבחירה האידיאלית כאן.

---

## 🛠️ הכנות: Debootstrap ו-QEMU

הכלי `debootstrap` מאפשר לנו ליצור מערכת קבצים של דביאן מאפס. עם זאת, מכיוון שהמחשב שעליו אנחנו עובדים מבוסס על ארכיטקטורת x86_64 ואנחנו מקמפלים למעבד ARM (ארכיטקטורת `armhf`), אנחנו חייבים להשתמש באמולטור.

נתקין את הכלים הנדרשים על המחשב המארח:

```bash
sudo apt update
sudo apt install debootstrap qemu-user-static
```

*(הערה: במערכות לינוקס מודרניות, QEMU משתלב ישירות בליבת המערכת ומבצע את האמולציה מאחורי הקלעים באופן שקוף. לכן אין צורך להעתיק את קובץ האמולטור ידנית לתוך התיקייה כפי שהיה נהוג בעבר).*

---

## 📥 שלב ראשון: הורדת קבצי הבסיס (First Stage)

בשלב הזה ניצור תיקייה ונמשוך אליה את קבצי הבסיס משרתי דביאן.
במדריך זה נשתמש בגרסת **Bookworm** (Debian 12), שהיא הגרסה היציבה והעדכנית ביותר.

```bash
mkdir rootfs
sudo debootstrap --arch=armhf --foreign bookworm rootfs
```

המתינו מספר דקות עד שההורדה והחילוץ של חבילות הבסיס יסתיימו.

---

## ⚙️ שלב שני: התקנה פנימית (Second Stage)

עכשיו ניכנס לתוך התיקייה שיצרנו בעזרת הפקודה `chroot`. פקודה זו מנחה את המערכת להתייחס לתיקייה החדשה כאילו היא כונן המערכת הראשי שלנו.

ניכנס לסביבה החדשה:

```bash
sudo chroot rootfs /bin/bash -i
```

כעת, כשאנחנו רצים "בתוך" סביבת ה-ARM המדומה, נפעיל את השלב השני של ההתקנה:

```bash
/debootstrap/debootstrap --second-stage
```

תהליך זה אורך כ-5 דקות. במהלכו, המערכת תתקין ותגדיר סופית את כל חבילות הבסיס.

---

## 🔧 הגדרות מערכת (Configuration)

כל עוד אנחנו עדיין בתוך סביבת ה-`chroot`, חובה עלינו להגדיר את המערכת כדי שתהיה מוכנה לעבודה עצמאית ברגע שנפעיל את כרטיס ה-DE10-Nano.

### 1. התקנת עורך טקסט
המערכת מגיעה ללא עורך טקסט נוח. נתקין את `nano` (ניתן להתקין גם `vim` במידה ותעדיפו):

```bash
apt install nano -y
```

### 2. הגדרת שם המארח (Hostname)
נגדיר שם שייצג את הכרטיס שלנו ברשת:

```bash
echo "de10-nano" > /etc/hostname
```

### 3. סיסמת Root
כדי שהמערכת לא תהיה חשופה ופרוצה, נגדיר סיסמה למשתמש הראשי:

```bash
passwd
```

### 4. עגינת מחיצות אוטומטית (fstab)
נגדיר למערכת לטעון את הכונן הראשי באופן אוטומטי בכל הפעלה. 
פתחו את הקובץ `/etc/fstab`:

```bash
nano /etc/fstab
```

והדביקו פנימה את השורות הבאות:

```text
none            /tmp    tmpfs   defaults,noatime,mode=1777  0   0
/dev/mmcblk0p2  /       ext4    defaults                    0   1
```

### 5. הפעלת חיבור סריאלי (Serial Console)
כדי שנוכל לראות את הודעות ה-Boot ולהתחבר ללוח עם כבל סריאלי (למשל דרך Putty):

```bash
systemctl enable serial-getty@ttyS0.service
```

### 6. הגדרות שפה ואזור (Locales)
נתקין ונגדיר את חבילת השפות (מומלץ לבחור מהרשימה את `en_US.UTF-8`):

```bash
apt install locales -y
dpkg-reconfigure locales
```

### 7. הגדרת רשת קווית (Ethernet)
כדי שהכרטיס ימשוך כתובת IP אוטומטית משרת ה-DHCP ברגע שנחבר אליו כבל רשת, פתחו את קובץ הגדרות הרשת:

```bash
nano /etc/network/interfaces
```

והדביקו את ההגדרות הבאות:
*(הערה: בלינוקס מודרני מנגנון השמות - Predictable Interface Names - השתנה לעיתים. במידה והחיבור נכשל, נסו להחליף בקובץ זה את המילה `eth0` ב-`end0`)*.

```text
auto lo eth0
iface lo inet loopback

allow-hotplug eth0
iface eth0 inet dhcp
```

### 8. עדכון מאגרי תוכנה (Sources.list)
נעדכן את רשימת המאגרים כך שיתאימו לגרסת Bookworm ויכללו גם מנהלי התקנים קנייניים במידת הצורך (non-free-firmware):

```bash
nano /etc/apt/sources.list
```

הדביקו פנימה:

```text
deb [http://deb.debian.org/debian/](http://deb.debian.org/debian/) bookworm main contrib non-free non-free-firmware
deb-src [http://deb.debian.org/debian/](http://deb.debian.org/debian/) bookworm main contrib non-free non-free-firmware
deb [http://deb.debian.org/debian/](http://deb.debian.org/debian/) bookworm-updates main contrib non-free non-free-firmware
deb-src [http://deb.debian.org/debian/](http://deb.debian.org/debian/) bookworm-updates main contrib non-free non-free-firmware
deb [http://security.debian.org/debian-security](http://security.debian.org/debian-security) bookworm-security main contrib non-free non-free-firmware
deb-src [http://security.debian.org/debian-security](http://security.debian.org/debian-security) bookworm-security main contrib non-free non-free-firmware
```

### 9. אפשרות חיבור מרחוק (SSH)
נתקין שרת SSH כדי שנוכל להתחבר לכרטיס בנוחות דרך הרשת:

```bash
apt update
apt install openssh-server -y
```

*(טיפ: אם ברצונכם לאפשר התחברות מרחוק כמשתמש root, פתחו את `/etc/ssh/sshd_config` ושנו את השורה `PermitRootLogin` ל-`yes`).*

### 10. התקנת כלי מערכת וספריות פיתוח
לסיום, נתקין מספר כלים חיוניים להמשך הפיתוח על הכרטיס:

```bash
apt install net-tools build-essential device-tree-compiler haveged -y
```

* **`net-tools`**: מאפשר שימוש בפקודות רשת בסיסיות כמו `ifconfig`.
* **`build-essential`**: מאפשר לקמפל תוכנות (GCC) ישירות על ה-DE10-Nano.
* **`device-tree-compiler`**: נדרש לצורך קימפול Device Trees וצריבת ה-FPGA ישירות מתוך סביבת הלינוקס.
* **`haveged`**: מאיץ את יצירת המספרים האקראיים במערכת, מה שמונע עיכובים משמעותיים בזמן העלייה של שרת ה-SSH.

---

## 📦 סיום: יצירת תמונת המערכת (Tarball)

סיימנו עם כל ההגדרות! כעת עלינו לנקות את שאריות ההתקנה ולארוז את כל המערכת שיצרנו לקובץ ארכיון אחד דחוס. קובץ זה יחולץ בהמשך ישירות לתוך כרטיס ה-SD.

נקו את המטמון וצאו מסביבת ה-chroot:

```bash
apt clean
exit
```

כעת חזרנו למערכת המארחת (נמצאים בתיקייה שמעל `rootfs`). נריץ את הפקודה ליצירת הארכיון:
> ⚠️ **זהירות:** שימו לב לנקודה `.` בסוף הפקודה הבאה! היא הכרחית כדי להנחות את הפקודה לארוז את כל התוכן הפנימי של התיקייה הנוכחית.

```bash
cd rootfs
sudo tar -cjpf ../rootfs.tar.bz2 .
cd ..
```

**זהו! המערכת המותאמת שלנו ארוזה ומוכנה בקובץ `rootfs.tar.bz2`.**
שימו לב שכאשר נחלץ את הקובץ הזה בהמשך לתוך כרטיס ה-SD, הוא יפרוס את כל הקבצים ישירות למחיצה מבלי ליצור תיקייה מקדימה, לכן חשוב לוודא שאתם נמצאים בתוך מחיצת ה-Root של ה-SD לפני ביצוע פעולת החילוץ.