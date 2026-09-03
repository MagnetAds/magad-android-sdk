<div dir="rtl">

# راهنمای یکپارچه‌سازی MagnetAd SDK در اندروید (ویژه ناشران)

این سند، راهنمای گام‌به‌گام استفاده از پکیج تبلیغاتی **MagnetAd** در پروژه‌های **اندروید (Kotlin/Java)** برای نمایش آگهی **تمام‌صفحه (Interstitial)** و **جایزه‌دار (Rewarded)** است.

---

## فهرست

1. [معرفی کوتاه](#۱-معرفی-کوتاه)
2. [پیش‌نیازها](#۲-پیشنیازها)
3. [گام ۱ — افزودن پکیج (فایل AAR)](#گام-۱--افزودن-پکیج-فایل-aar)
4. [گام ۲ — تنظیمات مانیفست و پروگارد](#گام-۲--تنظیمات-مانیفست-و-پروگارد)
5. [گام ۳ — دریافت شناسه‌ها (App Id و Placement Id)](#گام-۳--دریافت-شناسهها)
6. [گام ۴ — راه‌اندازی SDK](#گام-۴--راهاندازی-sdk)
7. [گام ۵ — بارگذاری و نمایش آگهی](#گام-۵--بارگذاری-و-نمایش-آگهی)
8. [گام ۶ — رویدادها (AdListener)](#گام-۶--رویدادها-adlistener-و-نتیجهٔ-بارگذاری-adloadresult)
9. [گام ۷ — آگهی ویدیویی و جایزه‌دار](#گام-۷--آگهی-ویدیویی-و-جایزهدار)
10. [کنترل صدای ویدیو](#کنترل-صدای-ویدیو)
11. [استفاده همزمان از چند جایگاه](#استفاده-همزمان-از-چند-جایگاه)
12. [مرجع کامل API](#مرجع-کامل-api)
13. [کدهای خطا](#کدهای-خطا)
14. [رفتار داخلی SDK](#رفتار-داخلی-sdk)
15. [عیب‌یابی مشکلات رایج](#عیبیابی-مشکلات-رایج)
16. [محدودیت‌ها](#محدودیتها)

---

## ۱. معرفی کوتاه

- نوع آگهی: **تمام‌صفحه (Interstitial)** و **جایزه‌دار (Rewarded)**، با خلاقیت **تصویری یا ویدیویی**.
- پلتفرم: **اندروید** (حداقل API 21 / Android 5.0).
- نوشته‌شده با **Kotlin** و **Coroutines**؛ از داخل کد Kotlin به‌سادگی و از Java نیز قابل استفاده است.
- بدون هیچ مجوز خطرناک و بدون دیالوگ درخواست دسترسی از کاربر.
- تمام متدهای عمومی **غیرمسدودکننده (non-blocking)** هستند؛ درخواست شبکه در پس‌زمینه انجام می‌شود و نتیجه روی **ترد اصلی** برمی‌گردد.
- الگوی استفاده مانند SDKهای استاندارد (AdMob / AppLovin) است: یک آگهی می‌سازید، در رویدادها گوش می‌دهید، سپس بارگذاری و نمایش می‌دهید.

> **نکته مهم درباره نوع خلاقیت:** شما تعیین نمی‌کنید که آگهی تصویری باشد یا ویدیویی. سرور بر اساس جایگاه تصمیم می‌گیرد و SDK به‌صورت خودکار پخش‌کننده درست را انتخاب می‌کند. کد شما در هر دو حالت یکسان است.

---

## ۲. پیش‌نیازها

<div dir="ltr">

| مورد | مقدار |
|------|-------|
| حداقل نسخه اندروید | **API 21** (Android 5.0) |
| compileSdk پروژه | **34** یا بالاتر |
| نسخه Kotlin پروژه | **1.9** یا بالاتر |
| زبان | **Kotlin** (پیشنهادی) یا Java |
| Java/JVM target | **11** |
| سیستم بیلد | **Gradle** (Kotlin DSL یا Groovy) |

</div>

> ‏compileSdk برنامه شما لازم نیست ۳۶ باشد. وابستگی‌های SDK طوری انتخاب شده‌اند که از **compileSdk 34 به بالا** کار کند، پس پروژه‌هایی که هنوز روی SDK قدیمی‌تر هستند مجبور به ارتقا نمی‌شوند.

مجوزهایی که پکیج به‌صورت خودکار (از طریق Manifest Merger) به برنامه شما اضافه می‌کند:

<div dir="ltr">

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="com.google.android.gms.permission.AD_ID" />
```

</div>

> نیازی نیست این مجوزها را دستی اضافه کنید؛ مانیفستِ پکیج آن‌ها را به مانیفست نهایی برنامه شما ادغام می‌کند.

> مجوز `AD_ID` از اندروید ۱۳ (API 33) به بعد برای خواندن شناسه تبلیغاتی گوگل لازم است. این مجوز خطرناک نیست و دیالوگی به کاربر نشان نمی‌دهد، ولی اگر در Google Play منتشر می‌کنید باید در فرم Data Safety اعلامش کنید.

---

## گام ۱ — افزودن پکیج (فایل AAR)

‏MagnetAd در حال حاضر روی **هیچ مخزن Maven عمومی** (Maven Central، JitPack و …) منتشر نمی‌شود. تنها مرجع رسمی توزیع، مخزن گیت‌هاب زیر است و نصب از طریق **فایل AAR** انجام می‌شود:

<div dir="ltr">

```
https://github.com/MagnetAds/magad-android-sdk
```

</div>

> یعنی `implementation("com.hasin:magnetad:…")` کار نمی‌کند و نباید هیچ مخزن Maven برای MagnetAd به `settings.gradle.kts` اضافه کنید. به‌جایش گام‌های زیر را دنبال کنید.

### ۱-۱) دریافت فایل AAR

فایل AAR را از مخزن بالا بگیرید (بخش **Releases** یا هر جایی که `README` آن مخزن اشاره می‌کند) و در پوشه `app/libs/` پروژه‌تان کپی کنید. اگر پوشه `libs` وجود ندارد بسازید:

<div dir="ltr">

```
<your-project>/
└── app/
    └── libs/
        └── magnetad-1.8.0.aar
```

</div>

> نام فایل شماره نسخه را در خود دارد. هنگام ارتقا، فایل قدیمی را **حذف کنید** و نام فایل را در `build.gradle.kts` به‌روز کنید؛ اگر هر دو فایل در `libs` بمانند، بیلد با خطای duplicate class شکست می‌خورد.

### ۱-۲) افزودن وابستگی و وابستگی‌های جانبی آن

چون AAR از مخزن Maven نمی‌آید، فایل POM کنارش وجود ندارد، پس **وابستگی‌های جانبی‌اش خودکار حل نمی‌شوند و باید دستی اضافه شوند**. در `build.gradle.kts` **ماژول برنامه** (app):

<div dir="ltr">

```kotlin
dependencies {
    implementation(files("libs/magnetad-1.8.0.aar"))

    // موردنیاز SDK. بدون این‌ها پروژه کامپایل می‌شود
    // ولی در زمان اجرا با NoClassDefFoundError کرش می‌کند:
    implementation("org.jetbrains.kotlin:kotlin-stdlib:2.0.21")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.8.1")
    implementation("com.google.android.gms:play-services-ads-identifier:18.1.0")
}
```

</div>

معادل Groovy (اگر از `build.gradle` استفاده می‌کنید):

<div dir="ltr">

```groovy
dependencies {
    implementation files('libs/magnetad-1.8.0.aar')

    implementation 'org.jetbrains.kotlin:kotlin-stdlib:2.0.21'
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1'
    implementation 'androidx.core:core-ktx:1.12.0'
    implementation 'com.squareup.okhttp3:okhttp:4.12.0'
    implementation 'com.squareup.okhttp3:logging-interceptor:4.12.0'
    implementation 'org.jetbrains.kotlinx:kotlinx-serialization-json:1.8.1'
    implementation 'com.google.android.gms:play-services-ads-identifier:18.1.0'
}
```

</div>

نکته‌های مهم درباره این نسخه‌ها:

- ‏`androidx.core:core-ktx` را روی **1.12.0** نگه دارید. نسخه‌های جدیدتر `compileSdk 36` می‌خواهند و پروژه شما را به ارتقا مجبور می‌کنند.
- اگر پروژه شما هرکدام از این کتابخانه‌ها را از قبل دارد (مستقیم یا از طریق SDK دیگری)، **دوباره اضافه نکنید**؛ Gradle خودش بالاترین نسخه را انتخاب می‌کند. فقط مطمئن شوید نسخه نهایی از موارد بالا قدیمی‌تر نباشد.
- ‏`play-services-ads-identifier` برای هدف‌گیری تبلیغات لازم است؛ اگر نگذارید، نه بیلد می‌شکند و نه کرش می‌شود، فقط دقت هدف‌گیری کم می‌شود.

### ۱-۳) مخزن‌ها برای وابستگی‌های جانبی

وابستگی‌های بالا از مخزن‌های عمومی می‌آیند، پس `settings.gradle.kts` شما باید آن‌ها را داشته باشد (معمولاً از قبل دارد):

<div dir="ltr">

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
    }
}
```

</div>

> اگر `dl.google.com` در دسترس نیست (مسدودسازی منطقه‌ای)، میرور `maven { url = uri("https://maven.myket.ir/") }` را **قبل از** `google()` اضافه کنید. توجه کنید این فقط میرور وابستگی‌های عمومی است و خود MagnetAd آنجا منتشر نمی‌شود.

---

## گام ۲ — تنظیمات مانیفست و پروگارد

### ۲-۱) مانیفست (خودکار)

پکیج موارد زیر را به‌صورت خودکار به مانیفست نهایی برنامه ادغام می‌کند و **نیازی به تنظیم دستی ندارید**:

- مجوزهای `INTERNET` و `ACCESS_NETWORK_STATE` و `AD_ID`.
- عنصر `<queries>` برای تشخیص نصب بودن مارکت‌ها (بازار / مایکت / گوگل‌پلی) در اندروید ۱۱ به بالا.

> ‏SDK هیچ `networkSecurityConfig` یا `usesCleartextTraffic` به مانیفست شما اضافه نمی‌کند. تمام ارتباطات روی `https` انجام می‌شود، پس تنظیمات امنیت شبکه برنامه شما دست‌نخورده می‌ماند و تداخلی در merge پیش نمی‌آید.

### ۲-۲) پروگارد / R8 (خودکار)

قوانین لازم برای حفظ کلاس‌های SDK از طریق `consumer-rules.pro` به‌صورت خودکار اعمال می‌شوند و **نیازی به افزودن قانون دستی نیست**.

---

## گام ۳ — دریافت شناسه‌ها

از پنل ناشر Magnet دو شناسه دریافت می‌کنید:

- ‏**App Id**: در `AdConfig` هنگام `initialize` استفاده می‌شود.
- ‏**Placement Id**: شناسه جایگاه؛ در `getInterstitialAd` استفاده می‌شود.

> این شناسه‌ها رشته (String) هستند و **اعتبارسنجی فرمت آن‌ها بر عهده سرور** است.

> نوع جایگاه (عادی یا جایزه‌دار) هنگام ساخت آن در پنل مشخص می‌شود، نه در کد شما. برای هر نوع، یک Placement Id جداگانه بگیرید.

---

## گام ۴ — راه‌اندازی SDK

‏SDK را **یک‌بار** و در کلاس `Application` مقداردهی اولیه کنید. این کار فوراً برمی‌گردد و هماهنگی با سرور در پس‌زمینه انجام می‌شود.

<div dir="ltr">

```kotlin
import android.app.Application
import android.util.Log
import com.hasin.magnetad.AdConfig
import com.hasin.magnetad.MagnetAdManager

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        MagnetAdManager.initialize(
            application = this,
            config = AdConfig(
                appId = "YOUR-APP-ID",
                debugMode = BuildConfig.DEBUG, // لاگ‌های SDK فقط در حالت دیباگ
                videoVolume = 1f               // صدای اولیه ویدیو، بین 0f تا 1f
            ),
            onInitComplete = { success ->
                // اختیاری: روی ترد اصلی صدا زده می‌شود
                Log.d("MagnetAd", "init success = $success")
            }
        )
    }

    override fun onTrimMemory(level: Int) {
        super.onTrimMemory(level)
        MagnetAdManager.onTrimMemory(level) // کمک به مدیریت حافظه کش تصاویر
    }
}
```

</div>

سپس کلاس Application خود را در مانیفست ثبت کنید:

<div dir="ltr">

```xml
<application
    android:name=".MyApplication"
    ... >
```

</div>

> ‏`getInterstitialAd` را می‌توانید حتی پیش از پایان هندشیک شبکه صدا بزنید، اما تا موفق شدن آن، بارگذاری آگهی ناموفق خواهد بود.

---

## گام ۵ — بارگذاری و نمایش آگهی

بارگذاری و نمایش کاملاً جدا و صریح‌اند: شما با `requestAd` آگهی را بارگذاری می‌کنید، در صورت موفقیت یک `adId` یک‌بارمصرف دریافت می‌کنید، و همان شناسه را هر وقت خواستید به `show` می‌دهید. SDK هرگز به‌خودی‌خود و بدون فراخوانی صریح شما آگهی بارگذاری نمی‌کند. قاعده طلایی: **آگهی را زودتر بارگذاری کنید، `adId` را نگه دارید، و در یک نقطه طبیعی بازی یا برنامه نمایش دهید.**

<div dir="ltr">

```kotlin
import android.os.Bundle
import android.util.Log
import androidx.activity.ComponentActivity
import com.hasin.magnetad.*

class MainActivity : ComponentActivity() {

    private var interstitialAd: InterstitialAd? = null
    private var cachedAdId: String? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // ۱) ساخت آگهی برای یک جایگاه مشخص
        interstitialAd = MagnetAdManager.getInterstitialAd(placementId = "YOUR-PLACEMENT-ID")

        // ۲) گوش دادن به رویدادهای مربوط به نمایش (بعد از show)
        interstitialAd?.setAdListener(object : AdListener() {
            override fun onAdShown() { /* آگهی روی صفحه آمد */ }
            override fun onAdClicked() { /* کاربر روی آگهی زد */ }
            override fun onAdDismissed() { /* کاربر آگهی را بست */ }

            override fun onAdFailedToShow(error: AdError) {
                Log.w("MagnetAd", "show failed: ${error.code}")
            }
        })

        // ۳) بارگذاری صریح؛ نتیجه فقط از طریق همین کال‌بک می‌آید
        interstitialAd?.requestAd(this) { result ->
            when (result) {
                is AdLoadResult.Success -> {
                    // آگهی آماده است؛ adId را نگه دارید و هر زمان خواستید نمایش دهید
                    cachedAdId = result.adId
                    cachedAdId?.let { interstitialAd?.show(this@MainActivity, it) }
                }
                is AdLoadResult.Failure -> {
                    Log.w("MagnetAd", "load failed: ${result.error.code}")
                }
            }
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        // ۴) آزادسازی منابع و لغو بارگذاریِ در جریان
        interstitialAd?.destroy()
        interstitialAd = null
    }
}
```

</div>

> **نمونه `InterstitialAd` را نگه دارید.** هر بار که `getInterstitialAd` را صدا بزنید یک نمونه **کاملاً تازه** با کش خالی ساخته می‌شود. اگر موقع نمایش دوباره `getInterstitialAd` را صدا بزنید، `show` با خطای `AD_NOT_READY` برمی‌گردد؛ چون `adId` متعلق به همان نمونه‌ای است که بارگذاری را انجام داده، نه به جایگاه.

> برای نمایش دوباره آگهی، دوباره `requestAd` را صدا بزنید و `adId` تازه‌ای بگیرید. هر `adId` فقط برای یک بار `show` معتبر است و به‌محض نمایش یا انقضا از رده خارج می‌شود.

---

## گام ۶ — رویدادها (AdListener) و نتیجهٔ بارگذاری (AdLoadResult)

نتیجه بارگذاری از طریق `AdListener` نمی‌آید؛ همان کال‌بکی که به `requestAd` می‌دهید یک `AdLoadResult` دریافت می‌کند:

<div dir="ltr">

| مقدار `AdLoadResult` | معنی |
|------------------------|------|
| `AdLoadResult.Success(adId)` | آگهی و فایل خلاقیتش (تصویر یا ویدیو) آماده‌اند؛ `adId` را نگه دارید و به `show` بدهید. |
| `AdLoadResult.Failure(error)` | بارگذاری ناموفق بود (شبکه، سرور، نبود موجودی و …). |

</div>

`AdListener` فقط رویدادهای **بعد از `show`** را گزارش می‌دهد:

<div dir="ltr">

| رویداد | زمان فراخوانی |
|--------|----------------|
| `onAdShown()` | آگهی روی صفحه ظاهر شد. |
| `onAdClicked()` | کاربر روی آگهی زد (هر آگهی یک بار). |
| `onAdDismissed()` | کاربر آگهی را بست. |
| `onAdFailedToShow(error)` | نمایش ممکن نشد (`adId` نامعتبر یا مصرف‌شده، انقضا، در دسترس نبودن Activity، خطای پخش ویدیو و …). |
| `onRewarded()` | فقط برای آگهی جایزه‌دار: کاربر ویدیو را تا آستانه لازم دید و جایزه باید داده شود. |

</div>

> تمام کال‌بک‌ها، چه کال‌بک `requestAd` و چه رویدادهای `AdListener`، روی **ترد اصلی (Main)** فراخوانی می‌شوند، پس می‌توانید مستقیماً UI را به‌روزرسانی کنید.

---

## گام ۷ — آگهی ویدیویی و جایزه‌دار

### تفاوت کد

**هیچ.** همان `getInterstitialAd` و `requestAd` و `show` را صدا می‌زنید. تنها کاری که برای جایگاه جایزه‌دار باید بکنید، override کردن `onRewarded()` است:

<div dir="ltr">

```kotlin
private lateinit var rewardedAd: InterstitialAd
private var rewardedAdId: String? = null

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)

    rewardedAd = MagnetAdManager.getInterstitialAd("YOUR-REWARDED-PLACEMENT-ID")

    rewardedAd.setAdListener(object : AdListener() {
        override fun onRewarded() {
            // کاربر ویدیو را کامل دید؛ اینجا جایزه را در UI نشان دهید
            showCoinAnimation()
        }

        override fun onAdDismissed() {
            rewardedAdId = null
            preloadRewarded() // بارگذاری بعدی
        }

        override fun onAdFailedToShow(error: AdError) {
            rewardedAdId = null
        }
    })

    preloadRewarded()
}

private fun preloadRewarded() {
    rewardedAd.requestAd(this) { result ->
        rewardedAdId = (result as? AdLoadResult.Success)?.adId
    }
}

private fun onWatchForCoinsClicked() {
    rewardedAdId?.let { rewardedAd.show(this, it) }
}
```

</div>

### تشخیص نوع جایگاه در کد

بعد از یک بارگذاری موفق، می‌توانید نوع خلاقیتی که سرور برگردانده را بخوانید:

<div dir="ltr">

```kotlin
when (rewardedAd.placementType) {
    PlacementType.REWARDED     -> { /* آگهی جایزه‌دار است */ }
    PlacementType.INTERSTITIAL -> { /* آگهی عادی است */ }
    null                       -> { /* هنوز چیزی بارگذاری نشده */ }
}
```

</div>

## کنترل صدای ویدیو

صدای اولیه را در `AdConfig` تعیین می‌کنید و هر زمان، حتی وسط پخش، می‌توانید تغییرش دهید:

<div dir="ltr">

```kotlin
MagnetAdManager.setVideoVolume(0.5f)   // بین 0f تا 1f؛ خارج از بازه clamp می‌شود
MagnetAdManager.getVideoVolume()       // مقدار فعلی

MagnetAdManager.setVideoMuted(true)    // بی‌صدا کردن بدون از دست دادن مقدار قبلی
MagnetAdManager.isVideoMuted()         // وضعیت فعلی
```

</div>

> ‏`setVideoMuted(false)` صدا را به همان مقداری برمی‌گرداند که پیش از بی‌صدا کردن تنظیم شده بود. این متدها را از هر تردی می‌توانید صدا بزنید.

> اگر بازی شما دکمه صدا دارد، وضعیت آن را هنگام `initialize` در `AdConfig.videoVolume` بدهید و بعد از آن با همین متدها هماهنگ نگه دارید. تغییر دادن مستقیم فیلد `AdConfig` بعد از `initialize` اثری ندارد.

---

## استفاده همزمان از چند جایگاه

اگر هم جایگاه عادی دارید و هم جایزه‌دار، برای هرکدام یک نمونه جدا بسازید و هر دو را به‌عنوان فیلد نگه دارید. نمونه‌ها کاملاً مستقل‌اند: هرکدام کش، کال‌بک و چرخه عمر خودش را دارد و درخواست همزمان هیچ تداخلی ایجاد نمی‌کند.

<div dir="ltr">

```kotlin
private lateinit var interstitialAd: InterstitialAd
private lateinit var rewardedAd: InterstitialAd

interstitialAd = MagnetAdManager.getInterstitialAd("PLACEMENT-INTERSTITIAL")
rewardedAd     = MagnetAdManager.getInterstitialAd("PLACEMENT-REWARDED")
```

</div>

> در هر لحظه فقط **یک** آگهی را نمایش دهید. SDK جلوی نمایش همزمان دو آگهی را نمی‌گیرد و نتیجه‌اش دو دیالوگ روی هم خواهد بود.

> در `onDestroy` روی هر دو نمونه `destroy()` را صدا بزنید.

---

## مرجع کامل API

<div dir="ltr">

```kotlin
object MagnetAdManager {
    fun initialize(
        application: Application,
        config: AdConfig,
        onInitComplete: ((success: Boolean) -> Unit)? = null
    )
    fun getInterstitialAd(placementId: String): InterstitialAd
    fun isInitialized(): Boolean
    fun getSDKVersion(): String
    fun getConfig(): AdConfig?

    fun setVideoVolume(volume: Float)
    fun getVideoVolume(): Float
    fun setVideoMuted(muted: Boolean)
    fun isVideoMuted(): Boolean

    fun clearCache()
    fun shutdown()
    fun onTrimMemory(level: Int)
}

class InterstitialAd {
    val state: AdState                  // IDLE / LOADING / LOADED / SHOWING / DESTROYED
    val placementType: PlacementType?   // بعد از بارگذاری موفق پر می‌شود

    fun setAdListener(listener: AdListener)
    fun isAdReady(): Boolean                                        // بدون نیاز به نگه داشتن adId توسط ناشر
    fun requestAd(
        context: Context,
        timeoutMs: Long = DEFAULT_REQUEST_TIMEOUT_MS,               // پیش‌فرض ۳۰ ثانیه؛ سقف کل بارگذاری
        callback: AdLoadCallback                                     // fun interface؛ از جاوا هم با لامبدا یا کلاس بی‌نام قابل استفاده است
    )                                                                // غیرمسدودکننده؛ نتیجه در callback
    fun show(activity: Activity, adId: String)                  // آگهیِ متناظر با adId را نمایش می‌دهد
    fun destroy()                                                  // لغو بارگذاری و آزادسازی منابع
}

abstract class AdListener {
    open fun onAdShown()
    open fun onAdDismissed()
    open fun onAdClicked()
    open fun onAdFailedToShow(error: AdError)
    open fun onRewarded()
}

sealed class AdLoadResult {
    data class Success(val adId: String) : AdLoadResult()
    data class Failure(val error: AdError) : AdLoadResult()
}

enum class PlacementType { INTERSTITIAL, REWARDED }

enum class AdState { IDLE, LOADING, LOADED, SHOWING, DESTROYED }

data class AdConfig(
    val appId: String,
    var debugMode: Boolean = false,
    val cacheSizeMb: Int = 50,
    var videoVolume: Float = 1f
)

data class AdError(val code: String, val message: String)
```

</div>

<div dir="ltr">

| متد | توضیح |
|-----|-------|
| `initialize(app, config, onInitComplete?)` | یک‌بار در `Application` صدا بزنید. غیرمسدودکننده. |
| `getInterstitialAd(placementId)` | یک نمونه **جدید** برای جایگاه موردنظر می‌سازد. پیش از `initialize` یا با شناسه خالی، exception می‌دهد. نمونه را نگه دارید و برای هر نمایش دوباره نسازید. |
| `state` | وضعیت لحظه‌ای نمونه؛ برای تشخیص «آگهی‌ای که فکر می‌کردم در حال نمایش است دیگر نیست» بدون انتظار برای کال‌بک. |
| `placementType` | نوع خلاقیتی که در آخرین بارگذاری موفق برگشته؛ پیش از آن `null`. |
| `isAdReady()` | آیا یک آگهیِ معتبر و منقضی‌نشده هم‌اکنون در کش آماده نمایش است. |
| `requestAd(context, timeoutMs?, callback)` | بارگذاری صریح آگهی در پس‌زمینه، با سقف زمانی کل (پیش‌فرض ۳۰ ثانیه؛ پس از آن `callback` با `AdErrorCode.TIMEOUT` صدا زده می‌شود). نتیجه فقط از طریق همین `callback` می‌آید. از هر تردی قابل فراخوانی است. SDK هیچ‌گاه خودکار بارگذاری نمی‌کند. اگر یک آگهیِ معتبر از قبل در کش باشد، بدون درخواست شبکه جدید همان `adId` بازگردانده می‌شود (کش تک‌ظرفیتی). اگر یک بارگذاری از قبل در جریان باشد (مثلاً دو بار لمس سریع دکمه)، `callback` این تماس هم در صف قرار می‌گیرد و همراه با همان بارگذاریِ درحال‌انجام نتیجه می‌گیرد. |
| `show(activity, adId)` | نمایش آگهیِ متناظر با `adId`. باید یک **Activity** فعال باشد. `adId` باید از آخرین `AdLoadResult.Success` باشد و هنوز مصرف نشده باشد. |
| `destroy()` | در `onDestroy` صدا بزنید تا منابع آزاد شوند. |
| `setVideoVolume(v)` / `setVideoMuted(b)` | کنترل صدای آگهی ویدیویی، حتی وسط پخش. |
| `shutdown()` | خاموش کردن کامل SDK (به‌ندرت لازم است). |

</div>

---

## کدهای خطا

این مقادیر در `AdError.code` قرار می‌گیرند:

<div dir="ltr">

| کد | معنی |
|----|------|
| `SDK_NOT_INITIALIZED` | پیش از `initialize` فراخوانی شده است. |
| `NETWORK_ERROR` | سرور در دسترس نبود یا خطای اتصال رخ داد. |
| `SERVER_ERROR` | خطای سمت سرور (وضعیت HTTP غیرموفق). |
| `INVALID_RESPONSE` | پاسخ سرور قابل پارس نبود. |
| `NO_FILL` | در این لحظه آگهی مناسبی برای این جایگاه موجود نبود. حالت عادی است، نه خطا. |
| `ASSET_LOAD_FAILED` | دانلود تصویر یا ویدیوی آگهی ناموفق بود. |
| `UNEXPECTED_AD_TYPE` | سرور نوع خلاقیتی برگرداند که این نسخه SDK پشتیبانی نمی‌کند. |
| `TIMEOUT` | بارگذاری از سقف زمانی `timeoutMs` گذشت. |
| `AD_NOT_READY` | `show` با `adId`ای صدا زده شد که با آگهی بارگذاری‌شده مطابقت ندارد؛ مثلاً هنوز `requestAd` صدا نزده‌اید، همان `adId` قبلاً نمایش داده شده، یا نمونه `InterstitialAd` را دوباره ساخته‌اید. |
| `AD_EXPIRED` | آگهیِ بارگذاری‌شده منقضی شده است؛ دوباره `requestAd` بزنید. |
| `ACTIVITY_UNAVAILABLE` | Activity در حال بسته شدن یا نابودی بود. |
| `SHOW_FAILED` | نمایش آگهی با خطا مواجه شد. |
| `VIDEO_PLAYBACK_ERROR` | پخش‌کننده ویدیو خطا داد. |
| `VIDEO_SERVER_DIED` | سرویس پخش‌کننده سیستم از کار افتاد. |
| `VIDEO_TIMEOUT` | پخش ویدیو گیر کرد و محافظ توقف آن را بست. |
| `UNKNOWN` | خطای نامشخص. |

</div>

> به‌جای تحلیل متن پیام، روی `error.code` شرط بگذارید (این مقادیر پایدارند).

---

## رفتار داخلی SDK

- **غیرمسدودکننده:** درخواست شبکه و دانلود خلاقیت در Coroutine و ترد پس‌زمینه انجام می‌شود؛ کال‌بک `requestAd` و رویدادهای `AdListener` روی ترد اصلی برمی‌گردند.
- **پیش‌بارگذاری کامل:** `AdLoadResult.Success` تنها زمانی صادر می‌شود که فایل خلاقیت (تصویر یا ویدیو) هم دانلود شده باشد، پس نمایش آنی و بدون صفحه سیاه است.
- **انتخاب خودکار نوع نمایش:** SDK از روی نوع فایلی که سرور اعلام می‌کند تصمیم می‌گیرد تصویر نشان دهد یا ویدیو. کد شما در هر دو حالت یکسان است.
- **بدون بارگذاری خودکار:** SDK هیچ‌وقت خودش آگهی بارگذاری نمی‌کند، نه در `initialize` و نه بعد از `show`. کنترل زمان بارگذاری و زمان نمایش کاملاً دست شماست و هر `adId` هم فقط یک بار قابل `show` شدن است.
- **دکمه بستن آگهی تصویری:** هنگام نمایش، دکمه بستن (✕) ابتدا یک **شمارش معکوس ۵ ثانیه‌ای** نشان می‌دهد و سپس فعال می‌شود؛ در این مدت دکمه سخت‌افزاری Back نیز غیرفعال است.
- **انقضای آگهی:** آگهیِ بارگذاری‌شده مدت اعتبار دارد؛ اگر دیر `show` بزنید، `AD_EXPIRED` می‌گیرید و باید دوباره بارگذاری کنید.
- **چرخه عمر و چرخش صفحه:** SDK وضعیت دیالوگ آگهی را در تغییرات پیکربندی (مثل چرخش صفحه) به‌صورت خودکار مدیریت می‌کند و کار اضافه‌ای لازم نیست.
- **مدیریت حافظه:** با فراخوانی `onTrimMemory` از `Application`، کش تصاویر در فشار حافظه آزاد می‌شود.

---

## عیب‌یابی مشکلات رایج

<div dir="ltr">

| نشانه | علت و راه‌حل |
|-------|--------------|
| `SDK_NOT_INITIALIZED` یا استثنا هنگام `getInterstitialAd` | `MagnetAdManager.initialize(...)` را در `Application.onCreate` صدا نزده‌اید یا کلاس Application در مانیفست ثبت نشده است. |
| بارگذاری همیشه با `NETWORK_ERROR` شکست می‌خورد | دسترسی اینترنت و در دسترس بودن سرور را بررسی کنید. |
| همیشه `NO_FILL` می‌گیرید | خطا نیست. اگر روی امولاتور تست می‌کنید، سرور معمولاً به امولاتور آگهی نمی‌دهد؛ روی دستگاه واقعی و با شناسه‌های تستی امتحان کنید. |
| `AD_NOT_READY` هنگام `show` | یا `adId` نامعتبر است، یا پیش از دریافت `AdLoadResult.Success` صدا زده‌اید، یا همان `adId` قبلاً مصرف شده، یا **نمونه `InterstitialAd` را دوباره ساخته‌اید**. نمونه را به‌عنوان فیلد نگه دارید. |
| `AD_EXPIRED` هنگام `show` | فاصله بین بارگذاری و نمایش طولانی شده است؛ در کال‌بک خطا دوباره `requestAd` بزنید. |
| `onRewarded` هرگز صدا زده نمی‌شود | یا جایگاه جایزه‌دار نیست (Placement Id را بررسی کنید)، یا کاربر ویدیو را تا آستانه ندیده است. با `debugMode = true` لاگ‌ها را ببینید. |
| ویدیو دیر شروع می‌شود | ویدیو پیش از نمایش کامل دانلود می‌شود، پس تأخیر در `requestAd` است نه در `show`. آگهی را زودتر بارگذاری کنید. |
| `VIDEO_TIMEOUT` می‌گیرید | پخش‌کننده گیر کرده است. اگر تکرار می‌شود، مدل دستگاه و فرمت ویدیو را به تیم Magnet گزارش دهید. |
| خطای `NoClassDefFoundError` هنگام اجرا | وابستگی‌های جانبی AAR را اضافه نکرده‌اید؛ فهرست کامل در گام ۱-۲ آمده است. |
| ‏Gradle نمی‌تواند وابستگی‌ها را دانلود کند | `dl.google.com` در دسترس نیست؛ میرور `maven { url = uri("https://maven.myket.ir/") }` را قبل از `google()` قرار دهید. |
| خطای `module was compiled with an incompatible version of Kotlin` | نسخه Kotlin پروژه شما قدیمی‌تر از 1.9 است؛ آن را به‌روزرسانی کنید. |
| کلاس‌های SDK پس از فعال کردن Minify حذف می‌شوند | معمولاً `consumer-rules.pro` این را پوشش می‌دهد؛ در صورت بروز مشکل، بیلد release را با لاگ بررسی کنید. |
| مارکت نصب‌شده باز نمی‌شود و به مرورگر می‌رود | بخش `<queries>` هنگام merge حذف شده است؛ خروجی merged manifest بیلد را بررسی کنید (باید سه پکیج مارکت را داشته باشد). |

</div>

برای دیباگ بهتر، در `AdConfig` مقدار `debugMode = true` قرار دهید تا لاگ‌های SDK با تگ `MagnetAd` در **logcat** چاپ شوند.

---

## محدودیت‌ها

- فقط آگهی **تمام‌صفحه (Interstitial)** و **جایزه‌دار (Rewarded)**. بنر و آگهی Native در این نسخه پشتیبانی نمی‌شوند.
- فرمت‌های ویدیویی پشتیبانی‌شده: **MP4** و **WebM**. پخش تطبیقی (HLS و DASH) پشتیبانی نمی‌شود.
- حداکثر حجم هر ویدیو: **۱۰۰ مگابایت**.
- در هر لحظه فقط یک آگهی قابل نمایش است.
- فقط **اندروید** (API 21 به بالا).

---

*نسخه SDK: `1.8.0` (مطابق مقدار `MagnetAdManager.getSDKVersion()`).*

</div>
