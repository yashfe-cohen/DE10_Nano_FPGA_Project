# קימפול ליבת המערכת (Linux Kernel)

[🔙 חזרה לתוכן העניינים של הלינוקס](../README.md)

במדריך זה נבנה את ליבת הלינוקס (Kernel) עבור כרטיס ה-DE10-Nano מאפס. ניקח את קוד המקור המעודכן, נגדיר את התצורה המותאמת ונרכיב את הכל בצורה נקייה ומסודרת.

---

## 🛠️ שלב ראשון: התקנת ספריות תלות

לפני שנוריד את קוד המקור, נדאג שלמחשב המארח יש את כל הכלים והספריות הנדרשים לקימפול הליבה.

הריצו את הפקודה הבאה בטרמינל:

```bash
sudo apt-get install libncurses-dev flex bison openssl libssl-dev dkms libelf-dev libudev-dev libpci-dev libiberty-dev libmpc-dev libgmp3-dev autoconf bc
```

---

## 📥 שלב שני: הורדת קוד המקור של הלינוקס

חברת Altera מתחזקת מזלג (Fork) ייעודי של הלינוקס התומך ברכיבי ה-SoC FPGA. נוריד את המאגר הרשמי ונבחר בגרסה מתקדמת ויציבה (`socfpga-6.7`) כדי להימנע מתקלות קוד ובעיות תאימות שהיו בגרסאות הישנות.

```bash
git clone https://github.com/altera-opensource/linux-socfpga.git

cd linux-socfpga

git checkout socfpga-6.7
```

---

## ⚙️ שלב שלישי: הגדרת תצורת הלינוקס (Kernel Configuration)

כדי שהלינוקס יכיר את החומרה של ה-DE10-Nano, ניצור תחילה קובץ הגדרות בסיסי מותאם.

```bash
make ARCH=arm socfpga_defconfig
```

כעת נפתח את ממשק ההגדרות הגרפי (`menuconfig`):

```bash
make ARCH=arm menuconfig
```

> 💡 **טיפ עבודה:**  
> בכל שלב שבו צריך לערוך קבצים או לנהל קוד, מומלץ לעבוד ישירות דרך **VS Code** באמצעות הפקודה:
>
> ```bash
> code .
> ```
>
> במקום להשתמש ב-`nano`.

### שינויים נדרשים בתפריט ההגדרות

#### ביטול הוספת גרסה אוטומטית לשם הלינוקס

נווטו אל:

```
General setup
```

ובטלו את האפשרות:

```
Automatically append version information to the version string
```

> פעולה זו מקלה על ניהול ובדיקת גרסאות שונות של דרייברים.

---

#### הפעלת תמיכה ב-Overlay Filesystem

נווטו אל:

```
File systems
```

והפעילו:

- Overlay filesystem support
- כל אפשרויות המשנה שלה

> **חשוב:** אפשרות זו נדרשת אם תרצו לצרוב את ה-FPGA מתוך לינוקס בזמן ריצה.

---

#### הפעלת CONFIGFS

נווטו אל:

```
File systems
    └── Pseudo filesystems
```

והפעילו את האפשרות:

```
Userspace-driven configuration filesystem
```

---

לאחר שסיימתם לבצע את כל ההגדרות:

- צאו מהתפריטים באמצעות **Exit**.
- אשרו את שמירת ההגדרות (**Save**).

---

## 🚀 שלב רביעי: קימפול ליבת המערכת

> ⚠️ **הערה חשובה מאוד:**  
> אין להשתמש ב־`sudo make` בזמן הקימפול.
>
> שימוש ב־`sudo` משנה את משתני הסביבה של ה-Cross Compiler ועלול לגרום לשגיאות קימפול.

### פתרון שגיאות Assembler נפוצות

אם במהלך הקימפול מתקבלות שגיאות מסוג **Assembler errors**, בצעו ניקוי מלא של סביבת הבנייה ולאחר מכן צרו מחדש את קובץ ההגדרות.

```bash
# ניקוי מלא של קבצי הקומפילציה
make mrproper

# יצירת קובץ ההגדרות מחדש
make ARCH=arm socfpga_defconfig
```

---

כעת התחילו את הקימפול המלא:

```bash
make ARCH=arm LOCALVERSION=zImage -j24
```

> ניתן לשנות את הערך של `-j24` בהתאם למספר הליבות במחשב המארח.

תהליך הקימפול נמשך בדרך כלל בין **5 ל-10 דקות**.

---

## ✅ סיום

בסיום הקימפול ייווצר קובץ ליבת לינוקס דחוס (`zImage`) המוכן לשימוש.

קובץ הפלט נמצא בנתיב:

```text
arch/arm/boot/zImage
```