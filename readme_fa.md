<div dir="rtl">

# راهنمای یکپارچه‌سازی MagnetAd SDK در اندروید (ویژه ناشران)

این سند، راهنمای گام‌به‌گام استفاده از پکیج تبلیغاتی **MagnetAd** در پروژه‌های **اندروید (Kotlin/Java)** برای نمایش آگهی **تمام‌صفحه (Interstitial)** است.

---

## فهرست

1. [معرفی کوتاه](#۱-معرفی-کوتاه)
2. [پیش‌نیازها](#۲-پیشنیازها)
3. [گام ۱ — افزودن پکیج (فایل AAR)](#گام-۱--افزودن-پکیج-فایل-aar)
4. [گام ۲ — تنظیمات مانیفست و پروگارد](#گام-۲--تنظیمات-مانیفست-و-پروگارد)
5. [گام ۳ — دریافت شناسه‌ها (App Id و Placement Id)](#گام-۳--دریافت-شناسهها)
6. [گام ۴ — راه‌اندازی SDK](#گام-۴--راهاندازی-sdk)
7. [گام ۵ — بارگذاری و نمایش آگهی](#گام-۵--بارگذاری-و-نمایش-آگهی)
8. [گام ۶ — رویدادها (AdListener) و نتیجهٔ بارگذاری (AdLoadResult)](#گام-۶--رویدادها-adlistener-و-نتیجهٔ-بارگذاری-adloadresult)
9. [مرجع کامل API](#مرجع-کامل-api)
10. [کدهای خطا](#کدهای-خطا)
11. [رفتار داخلی SDK](#رفتار-داخلی-sdk)
12. [عیب‌یابی مشکلات رایج](#عیبیابی-مشکلات-رایج)
13. [محدودیت‌ها](#محدودیتها)

---

## ۱. معرفی کوتاه

- نوع آگهی: **تمام‌صفحه (Interstitial)** — تصویری.
- پلتفرم: **اندروید** (حداقل API 21 / Android 5.0).
- نوشته‌شده با **Kotlin** و **Coroutines**؛ از داخل کد Kotlin به‌سادگی و از Java نیز قابل استفاده است.
- شبکه با **OkHttp** و پارس پاسخ با **kotlinx.serialization** انجام می‌شود (چون توزیع از طریق فایل AAR است، این وابستگی‌ها را باید خودتان به `build.gradle` اضافه کنید — [گام ۱](#گام-۱--افزودن-پکیج-فایل-aar)).
- بدون هیچ مجوز خطرناک (dangerous) و بدون دیالوگ درخواست دسترسی از کاربر؛ تنها مجوزهای عادی `INTERNET`، `ACCESS_NETWORK_STATE` و `AD_ID` به مانیفست اضافه می‌شوند.
- تمام متدهای عمومی **غیرمسدودکننده (non-blocking)** هستند؛ درخواست شبکه در پس‌زمینه انجام می‌شود و نتیجه روی **ترد اصلی** برمی‌گردد: نتیجهٔ بارگذاری از طریق کال‌بکِ خودِ `requestAd` (یک `AdLoadResult`) و رویدادهای پس از نمایش از طریق `AdListener`.
- الگوی استفاده مانند SDKهای استاندارد (AdMob / AppLovin) است: یک آگهی می‌سازید، در رویدادها گوش می‌دهید، سپس بارگذاری و نمایش می‌دهید.

---

## ۲. پیش‌نیازها

<div dir="ltr">

| ⁧مورد⁩ | ⁧مقدار⁩ |
|------|-------|
| ⁧حداقل نسخهٔ اندروید⁩ | ⁦**API 21** (Android 5.0)⁩ |
| ⁧compileSdk پروژه⁩ | ⁧**34** یا بالاتر⁩ |
| ⁧نسخهٔ Kotlin پروژه⁩ | ⁧**1.9** یا بالاتر⁩ |
| ⁧زبان⁩ | ⁧**Kotlin** (پیشنهادی) یا Java⁩ |
| ⁦Java/JVM target⁩ | ⁦**11**⁩ |
| ⁧سیستم بیلد⁩ | ⁧**Gradle** (Kotlin DSL یا Groovy)⁩ |

</div>

> ‏compileSdk برنامهٔ شما لازم نیست حتماً ۳۶ باشد؛ وابستگی‌های SDK طوری انتخاب شده‌اند که با **compileSdk 34 به بالا** کار کنند تا پروژه‌هایی که هنوز روی SDK قدیمی‌تر هستند مجبور به ارتقا نشوند.

مجوزها و تنظیماتی که پکیج به‌صورت خودکار (از طریق Manifest Merger) به برنامهٔ شما اضافه می‌کند:

<div dir="ltr">

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="com.google.android.gms.permission.AD_ID" />
```

</div>

> نیازی نیست این مجوزها را دستی اضافه کنید؛ مانیفستِ پکیج آن‌ها را به مانیفست نهایی برنامهٔ شما ادغام می‌کند. هیچ‌کدام **مجوز خطرناک (dangerous)** نیستند و هیچ دیالوگ درخواست دسترسی به کاربر نشان داده نمی‌شود.

### دربارهٔ مجوز `AD_ID`

‏SDK برای هدف‌گیری بهتر آگهی، مجوز `com.google.android.gms.permission.AD_ID` را به مانیفست شما اضافه می‌کند. نکات لازم:

- اگر برنامه را در **Google Play** منتشر می‌کنید، در بخش *Data safety* کنسول پلی باید استفاده از Advertising ID را اعلام کنید (برای انتشار در **مایکت / بازار** چنین الزامی وجود ندارد).
- اگر برنامهٔ شما مخاطب **کودکان** دارد یا به هر دلیل نمی‌خواهید این مجوز در برنامه‌تان باشد، می‌توانید آن را در مانیفست خود حذف کنید:

<div dir="ltr">

```xml
<uses-permission android:name="com.google.android.gms.permission.AD_ID"
    tools:node="remove" />
```

</div>

> با حذف این مجوز، SDK همچنان کار می‌کند و آگهی نمایش داده می‌شود؛ فقط دقت هدف‌گیری کمتر خواهد بود. برای استفاده از `tools:node` فراموش نکنید `xmlns:tools="http://schemas.android.com/tools"` را به تگ `<manifest>` اضافه کنید.

---

## گام ۱ — افزودن پکیج (فایل AAR)

‏MagnetAd فعلاً روی هیچ مخزن Maven عمومی (Maven Central / jitpack / …) منتشر نمی‌شود. تنها مرجع رسمی انتشار، مخزن گیت‌هاب زیر است و نصب از طریق **فایل AAR** انجام می‌شود:

<div dir="ltr">

```
https://github.com/MagnetAds/magad-android-sdk
```

</div>

> یعنی `implementation("com.hasin:magnetad:…")` کار نمی‌کند و نباید مخزن Maven ای برای MagnetAd به `settings.gradle.kts` اضافه کنید. مراحل زیر را دنبال کنید.

### ۱-۱) دریافت فایل AAR

فایل AAR را از مخزن بالا بردارید (بخش **Releases** یا مسیری که در `README` مخزن اعلام شده) و در پوشهٔ `app/libs/` پروژهٔ خود کپی کنید. اگر پوشهٔ `libs` وجود ندارد، بسازید:

<div dir="ltr">

```
<your-project>/
└── app/
    └── libs/
        └── magnetad-1.0.0.aar
```

</div>

> نام فایل شمارهٔ نسخه را در خود دارد. هنگام ارتقا، فایل قدیمی را **حذف** کنید و نام فایل جدید را در `build.gradle.kts` به‌روزرسانی کنید؛ اگر هر دو فایل در `libs` بمانند، بیلد با خطای کلاس تکراری (duplicate class) شکست می‌خورد.

### ۱-۲) افزودن وابستگی و وابستگی‌های جانبی

چون AAR از مخزن Maven نمی‌آید، فایل POM ی هم همراهش نیست؛ بنابراین **وابستگی‌های جانبی به‌صورت خودکار آورده نمی‌شوند و باید دستی اضافه شوند**. در `build.gradle.kts` **ماژول برنامه** (app):

<div dir="ltr">

```kotlin
dependencies {
    implementation(files("libs/magnetad-1.0.0.aar"))

    // Required by the SDK. Without these the project still compiles,
    // but crashes at runtime with NoClassDefFoundError:
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
    implementation files('libs/magnetad-1.0.0.aar')

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

نکات مهم دربارهٔ این نسخه‌ها:

- ‏`androidx.core:core-ktx` را روی **1.12.0** نگه دارید. نسخه‌های جدیدتر `compileSdk 36` را الزامی می‌کنند و پروژهٔ شما را مجبور به ارتقا می‌کنند.
- اگر پروژهٔ شما از قبل هر کدام از این کتابخانه‌ها را (مستقیم یا از طریق یک SDK دیگر) دارد، آن را **دوباره اضافه نکنید**؛ Gradle خودش بالاترین نسخه را انتخاب می‌کند. فقط مطمئن شوید نسخهٔ نهایی از موارد بالا قدیمی‌تر نباشد.
- ‏`play-services-ads-identifier` برای هدف‌گیری آگهی لازم است؛ اگر آن را اضافه نکنید، SDK هنگام ساخت خطا نمی‌دهد و در زمان اجرا هم کرش نمی‌کند، اما دقت هدف‌گیری کمتر خواهد بود.

### ۱-۳) مخزن‌ها برای دانلود وابستگی‌های جانبی

وابستگی‌های بالا از مخزن‌های عمومی می‌آیند، پس `settings.gradle.kts` شما باید آن‌ها را داشته باشد (این‌ها معمولاً از قبل در پروژه هستند):

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

> اگر `dl.google.com` در دسترس نیست (مسدودسازی منطقه‌ای)، میرور `maven { url = uri("https://maven.myket.ir/") }` را **قبل از** `google()` اضافه کنید. توجه کنید این فقط یک میرور برای وابستگی‌های عمومی است و خودِ MagnetAd روی آن منتشر نمی‌شود.

---

## گام ۲ — تنظیمات مانیفست و پروگارد

### ۲-۱) مانیفست (خودکار)

پکیج موارد زیر را به‌صورت خودکار به مانیفست نهایی برنامه ادغام می‌کند و **نیازی به تنظیم دستی ندارید**:

- مجوزهای `INTERNET` و `ACCESS_NETWORK_STATE` و `com.google.android.gms.permission.AD_ID` (هیچ‌کدام خطرناک نیستند — به بخش [دربارهٔ مجوز `AD_ID`](#دربارهٔ-مجوز-ad_id) مراجعه کنید).
- عنصر `<queries>` برای تشخیص نصب‌بودن مارکت‌ها (بازار / مایکت / گوگل‌پلی) در اندروید ۱۱ به بالا.

و همین. مانیفست پکیج **هیچ تگ `<application>` ندارد** و بنابراین هیچ‌کدام از تنظیمات برنامهٔ شما را بازنویسی نمی‌کند.

> **تغییر نسبت به نسخه‌های پیشین:** پکیج دیگر `android:usesCleartextTraffic="true"` و `networkSecurityConfig` را به مانیفست شما اضافه **نمی‌کند**؛ ارتباط با سرویس تماماً روی `https` است. نتیجهٔ عملی برای شما: اگر برنامه‌تان `networkSecurityConfig` سفارشی دارد، دیگر هیچ تداخلی در Manifest Merger رخ نمی‌دهد و نیازی به `tools:replace` نیست.

### ۲-۲) پروگارد / R8 (خودکار)

قوانین لازم از طریق `consumer-rules.pro` که داخل خودِ AAR بسته‌بندی شده به‌صورت خودکار اعمال می‌شوند؛ **نیازی به افزودن قانون دستی نیست**. این قوانین شامل موارد زیر است:

- حفظ کلاس‌های عمومی SDK (`MagnetAdManager`، `InterstitialAd`، `AdConfig`، `AdError`، `AdListener`، `AdLoadResult`، `AdLoadCallback`).
- قوانین `kotlinx.serialization` — بدون این‌ها، در بیلد release که minify فعال است، پاسخ سرور قابل پارس نمی‌شد و همهٔ درخواست‌ها با `INVALID_RESPONSE` شکست می‌خوردند.
- قوانین `-dontwarn` برای OkHttp/Okio و ارائه‌دهنده‌های اختیاری TLS (Conscrypt / BouncyCastle / OpenJSSE) — این‌ها کلاس‌هایی هستند که OkHttp فقط در صورت حضورشان استفاده می‌کند؛ بدون این قوانین، نبودشان در بیلد release شما به خطای «missing class» تبدیل می‌شد.

> اگر بیلد release شما با خطای R8 مربوط به یکی از پکیج‌های بالا شکست خورد، یعنی قوانین AAR اعمال نشده‌اند؛ ابتدا مطمئن شوید AAR/آرتیفکت را از نسخهٔ به‌روز برداشته‌اید.

---

## گام ۳ — دریافت شناسه‌ها

از پنل ناشر Magnet دو شناسه دریافت می‌کنید:

- ‏**App Id** — در `AdConfig` هنگام `initialize` استفاده می‌شود.
- ‏**Placement Id** — شناسهٔ جایگاه/Placement؛ در `getInterstitialAd` استفاده می‌شود.

> این شناسه‌ها رشته (String) هستند و **اعتبارسنجی فرمت آن‌ها بر عهدهٔ سرور** است.

> در تمام مراحل توسعه و انتشار از همین شناسه‌های دریافتی از پنل استفاده کنید.

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
                debugMode = BuildConfig.DEBUG // SDK logs only in debug builds
            ),
            onInitComplete = { success ->
                // Optional; invoked on the main thread
                Log.d("MagnetAd", "init success = $success")
            }
        )
    }

    override fun onTrimMemory(level: Int) {
        super.onTrimMemory(level)
        MagnetAdManager.onTrimMemory(level) // helps release the image cache under memory pressure
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

> ‏`getInterstitialAd` را می‌توانید حتی پیش از پایان هندشیک شبکه صدا بزنید، اما تا موفق‌شدن آن، بارگذاری آگهی ناموفق خواهد بود.

> **اگر `onInitComplete` مقدار `false` داد** (مثلاً کاربر هنگام اجرا اینترنت نداشته)، می‌توانید بعداً دوباره `initialize` را با همان پارامترها صدا بزنید؛ فراخوانی دوم واقعاً هندشیک را از نو تلاش می‌کند. فقط هندشیکِ **موفق** باعث می‌شود فراخوانی‌های بعدی نادیده گرفته شوند، پس تکرار بی‌ضرر است و لازم نیست خودتان وضعیت را نگه دارید.

---

## گام ۵ — بارگذاری و نمایش آگهی

بارگذاری و نمایش کاملاً جدا و صریح‌اند: شما با `requestAd` آگهی را بارگذاری می‌کنید، در صورت موفقیت یک `adId` یک‌بارمصرف دریافت می‌کنید، و همان شناسه را هر وقت خواستید به `show` می‌دهید. SDK هرگز به‌خودی‌خود و بدون فراخوانی صریح شما آگهی بارگذاری نمی‌کند. قاعدهٔ طلایی: **آگهی را زودتر بارگذاری کنید، `adId` را نگه دارید، و در یک نقطهٔ طبیعی بازی/برنامه نمایش دهید.**

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

        // 1) Create an ad object for a specific placement
        interstitialAd = MagnetAdManager.getInterstitialAd(placementId = "YOUR-PLACEMENT-ID")

        // 2) Listen to show-related events (everything after show)
        interstitialAd?.setAdListener(object : AdListener() {
            override fun onAdShown() { /* the ad appeared on screen */ }
            override fun onAdClicked() { /* the user tapped the ad */ }
            override fun onAdDismissed() { /* the ad was closed */ }

            override fun onAdFailedToShow(error: AdError) {
                Log.w("MagnetAd", "show failed: ${error.code}")
            }
        })

        // 3) Explicit load; the result arrives only through this callback
        interstitialAd?.requestAd(this) { result ->
            when (result) {
                is AdLoadResult.Success -> {
                    // The ad is ready; keep the adId and show it whenever you want
                    cachedAdId = result.adId
                    cachedAdId?.let { interstitialAd?.show(this@MainActivity, it) }
                }
                is AdLoadResult.Failure -> {
                    Log.w("MagnetAd", "load failed: ${result.error.code} — ${result.error.message}")
                }
            }
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        // 4) Release resources and cancel any in-flight load
        interstitialAd?.destroy()
        interstitialAd = null
    }
}
```

</div>

> برای نمایش دوبارهٔ آگهی، دوباره `requestAd` را صدا بزنید و `adId` تازه‌ای بگیرید. هر `adId` فقط برای یک بار `show` معتبر است — به‌محض نمایش (یا انقضا) از رده خارج می‌شود، پس هر نمونهٔ `InterstitialAd` پس از هر نمایش نیاز به بارگذاری مجدد دارد.

### همین کد در Java

‏API عمومی SDK برای مصرف از Java آماده شده است: متدهای `MagnetAdManager` به‌صورت `static` در دسترس‌اند (نیازی به `.INSTANCE` نیست)، پارامترهای پیش‌فرض overloadهای جاوایی دارند، `AdListener` یک **کلاس abstract** با پیاده‌سازی‌های خالی است (پس فقط همان رویدادی را که لازم دارید override می‌کنید) و `AdLoadCallback` یک **فانکشنال‌اینترفیس** است، پس با لامبدای جاوا کار می‌کند.

<div dir="ltr">

```java
import android.app.Activity;
import android.os.Bundle;
import android.util.Log;
import androidx.annotation.Nullable;
import androidx.appcompat.app.AppCompatActivity;
import com.hasin.magnetad.AdError;
import com.hasin.magnetad.AdListener;
import com.hasin.magnetad.AdLoadResult;
import com.hasin.magnetad.InterstitialAd;
import com.hasin.magnetad.MagnetAdManager;

public class MainActivity extends AppCompatActivity {

    private InterstitialAd interstitialAd;
    private String cachedAdId;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        interstitialAd = MagnetAdManager.getInterstitialAd("YOUR-PLACEMENT-ID");

        interstitialAd.setAdListener(new AdListener() {
            @Override
            public void onAdShown() { /* the ad appeared on screen */ }

            @Override
            public void onAdDismissed() { /* resume the game/app */ }

            @Override
            public void onAdFailedToShow(AdError error) {
                Log.w("MagnetAd", "show failed: " + error.getCode());
                // Resume here too, exactly as in onAdDismissed
            }
        });

        interstitialAd.requestAd(this, result -> {
            if (result instanceof AdLoadResult.Success) {
                cachedAdId = ((AdLoadResult.Success) result).getAdId();
                interstitialAd.show(MainActivity.this, cachedAdId);
            } else {
                AdError error = ((AdLoadResult.Failure) result).getError();
                Log.w("MagnetAd", "load failed: " + error.getCode());
            }
        });
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (interstitialAd != null) {
            interstitialAd.destroy();
            interstitialAd = null;
        }
    }
}
```

</div>

و مقداردهی اولیه در کلاس `Application`:

<div dir="ltr">

```java
MagnetAdManager.initialize(this, new AdConfig("YOUR-APP-ID", BuildConfig.DEBUG));

// Or with a handshake-completion callback. The parameter is a Kotlin function
// type, so the Java lambda must return Unit.INSTANCE:
MagnetAdManager.initialize(this, new AdConfig("YOUR-APP-ID", BuildConfig.DEBUG), success -> {
    Log.d("MagnetAd", "init success = " + success);
    return kotlin.Unit.INSTANCE;
});
```

</div>

> برای استفاده از لامبدا در Java به `compileOptions` با `JavaVersion.VERSION_11` (یا حداقل ۸) نیاز دارید — همان چیزی که در [پیش‌نیازها](#۲-پیشنیازها) آمده است.

---

## گام ۶ — رویدادها (AdListener) و نتیجهٔ بارگذاری (AdLoadResult)

نتیجهٔ بارگذاری دیگر از طریق `AdListener` نمی‌آید؛ همان کال‌بکی که به `requestAd` می‌دهید یک `AdLoadResult` دریافت می‌کند:

<div dir="ltr">

| ⁧مقدار `AdLoadResult`⁩ | ⁧معنی⁩ |
|------------------------|------|
| ⁦`AdLoadResult.Success(adId)`⁩ | ⁧آگهی و تصویرش آماده‌اند؛ `adId` را نگه دارید و به `show` بدهید.⁩ |
| ⁦`AdLoadResult.Failure(error)`⁩ | ⁧بارگذاری ناموفق بود (شبکه، سرور، عدم موجودی و …).⁩ |

</div>

‏`AdListener` فقط رویدادهای **بعد از `show`** را گزارش می‌دهد:

<div dir="ltr">

| ⁧رویداد⁩ | ⁧زمان فراخوانی⁩ |
|--------|----------------|
| ⁦`onAdShown()`⁩ | ⁧آگهی روی صفحه ظاهر شد.⁩ |
| ⁦`onAdClicked()`⁩ | ⁧کاربر روی تصویر آگهی زد (هر آگهی یک بار). **بلافاصله پس از آن آگهی بسته می‌شود** و `onAdDismissed()` هم صدا زده می‌شود.⁩ |
| ⁦`onAdDismissed()`⁩ | ⁧آگهی بسته شد — چه با دکمهٔ ✕ و چه به‌دنبال کلیک کاربر.⁩ |
| ⁦`onAdFailedToShow(error)`⁩ | ⁧نمایش ممکن نشد (`adId` نامعتبر/مصرف‌شده، انقضا، در دسترس نبودن Activity، یا خطا در رندر تصویر).⁩ |

</div>

> **مهم برای بازی‌ها:** اگر ادامهٔ بازی/برنامه را به `onAdDismissed()` گره زده‌اید، حتماً همان کار را در `onAdFailedToShow(error)` هم انجام دهید. در حالت خطای نمایش، آگهی به‌صورت خودکار بسته می‌شود اما **فقط** `onAdFailedToShow` صدا زده می‌شود و `onAdDismissed` نمی‌آید؛ بدون این کار، برنامهٔ شما در آن حالت معلق می‌ماند.

> تمام کال‌بک‌ها — چه کال‌بک `requestAd` و چه رویدادهای `AdListener` — روی **ترد اصلی (Main)** فراخوانی می‌شوند، پس می‌توانید مستقیماً UI را به‌روزرسانی کنید.

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
    fun clearCache()
    fun shutdown()
    fun onTrimMemory(level: Int)
}

class InterstitialAd {
    fun setAdListener(listener: AdListener)
    fun isAdReady(): Boolean                                  // no need to hold on to the adId yourself
    fun requestAd(
        context: Context,
        callback: AdLoadCallback                              // fun interface; usable from Java as a lambda
    )                                                         // non-blocking; result arrives in the callback
                                                              // the whole load is capped at 15s -> TIMEOUT
    fun show(activity: Activity, adId: String)                // shows the ad matching this adId
    fun destroy()                                             // cancels the load and releases resources
}

sealed class AdLoadResult {
    data class Success(val adId: String) : AdLoadResult()
    data class Failure(val error: AdError) : AdLoadResult()
}

data class AdConfig(
    val appId: String,
    var debugMode: Boolean = false,
    val cacheSizeMb: Int = 50
)

data class AdError(val code: String, val message: String)
```

</div>

<div dir="ltr">

| ⁧متد⁩ | ⁧توضیح⁩ |
|-----|-------|
| ⁦`initialize(app, config, onInitComplete?)`⁩ | ⁧یک‌بار در `Application` صدا بزنید. غیرمسدودکننده.⁩ |
| ⁦`getInterstitialAd(placementId)`⁩ | ⁧یک آگهیِ تمام‌صفحه برای جایگاه موردنظر می‌سازد. پیش از `initialize` یا با شناسهٔ خالی، exception می‌دهد.⁩ |
| ⁦`isAdReady()`⁩ | ⁧آیا یک آگهیِ معتبر و منقضی‌نشده هم‌اکنون در کش آماده نمایش است — بدون نیاز به نگه‌داشتن `adId` توسط ناشر.⁩ |
| ⁦`requestAd(context, callback)`⁩ | ⁧بارگذاری صریح آگهی در پس‌زمینه، با سقف زمانی کل و ثابتِ ۱۵ ثانیه؛ پس از آن `callback` با `AdErrorCode.TIMEOUT` صدا زده می‌شود. این سقف قابل تنظیم نیست. نتیجه (`AdLoadResult`) فقط از طریق همین `callback` می‌آید. از هر تردی قابل فراخوانی است. SDK هیچ‌گاه خودکار بارگذاری نمی‌کند. اگر یک آگهیِ معتبر از قبل در کش باشد، بدون درخواست شبکهٔ جدید همان `adId` بازگردانده می‌شود (کش تک‌ظرفیتی). اگر یک بارگذاری از قبل در حال انجام باشد (مثلاً دو‌بار لمس سریع دکمه)، `callback` این تماس هم در صف قرار می‌گیرد و همراه با همان بارگذاریِ درحال‌انجام نتیجه می‌گیرد — نه اینکه نادیده گرفته شود.⁩ |
| ⁦`show(activity, adId)`⁩ | ⁧نمایش آگهیِ متناظر با `adId`. باید یک **Activity** فعال باشد. `adId` باید از آخرین `AdLoadResult.Success` باشد و هنوز مصرف نشده باشد.⁩ |
| ⁦`destroy()`⁩ | ⁧در `onDestroy` صدا بزنید تا منابع آزاد شوند.⁩ |
| ⁦`shutdown()`⁩ | ⁧خاموش‌کردن کامل SDK (به‌ندرت لازم است).⁩ |

</div>

---

## کدهای خطا

این مقادیر در `AdError.code` قرار می‌گیرند:

<div dir="ltr">

| ⁧کد⁩ | ⁧معنی⁩ |
|----|------|
| ⁦`SDK_NOT_INITIALIZED`⁩ | ⁧پیش از `initialize` فراخوانی شده است.⁩ |
| ⁦`NETWORK_ERROR`⁩ | ⁧سرور در دسترس نبود یا خطای اتصال رخ داد.⁩ |
| ⁦`SERVER_ERROR`⁩ | ⁧خطای سمت سرور (وضعیت HTTP غیرموفق).⁩ |
| ⁦`INVALID_RESPONSE`⁩ | ⁧پاسخ سرور قابل پارس نبود.⁩ |
| ⁦`ASSET_LOAD_FAILED`⁩ | ⁧دانلود تصویر آگهی ناموفق بود.⁩ |
| ⁦`NO_FILL`⁩ | ⁧در این لحظه آگهی مناسبی برای نمایش وجود نداشت. خطا نیست و در کار عادی رخ می‌دهد؛ بی‌سروصدا از آن عبور کنید و بعداً دوباره `requestAd` بزنید.⁩ |
| ⁦`TIMEOUT`⁩ | ⁧کل بارگذاری (درخواست شبکه + دانلود تصویر) از سقف زمانی ثابتِ ۱۵ ثانیه گذشت.⁩ |
| ⁦`AD_NOT_READY`⁩ | ⁧`show` با `adId`ای صدا زده شد که با آگهی بارگذاری‌شده مطابقت ندارد — مثلاً هنوز `requestAd` صدا نزده‌اید، یا همان `adId` قبلاً یک بار نمایش داده شده است.⁩ |
| ⁦`AD_EXPIRED`⁩ | ⁧آگهیِ بارگذاری‌شده منقضی شده است؛ دوباره `requestAd` بزنید.⁩ |
| ⁦`ACTIVITY_UNAVAILABLE`⁩ | ⁧Activity در حال بسته‌شدن/نابودی بود.⁩ |
| ⁦`SHOW_FAILED`⁩ | ⁧نمایش آگهی با خطا مواجه شد.⁩ |
| ⁦`UNKNOWN`⁩ | ⁧خطای نامشخص.⁩ |

</div>

> به‌جای تحلیل متن پیام، روی `error.code` شرط بگذارید (این مقادیر پایدارند).

---

## رفتار داخلی SDK

- **غیرمسدودکننده:** درخواست شبکه و دانلود تصویر در Coroutine و ترد پس‌زمینه انجام می‌شود؛ کال‌بک `requestAd` و رویدادهای `AdListener` روی ترد اصلی برمی‌گردند.
- **پیش‌بارگذاری تصویر:** `AdLoadResult.Success` تنها زمانی صادر می‌شود که **تصویر آگهی نیز دانلود و دیکود شده باشد**، پس نمایش، آنی و بدون صفحهٔ سیاه است.
- **بدون بارگذاری خودکار:** SDK هیچ‌وقت خودش آگهی بارگذاری نمی‌کند — نه در `initialize` و نه بعد از `show`. کنترل زمان بارگذاری و زمان نمایش کاملاً دست شماست؛ هر `adId` هم فقط یک بار قابل `show`شدن است.
- **کش تک‌ظرفیتی:** در هر لحظه حداکثر یک آگهی بارگذاری‌شده در کش وجود دارد. اگر `requestAd` را دوباره صدا بزنید درحالی‌که یک آگهیِ معتبر (نمایش‌داده‌نشده و منقضی‌نشده) از قبل در کش است، درخواست شبکهٔ جدیدی زده نمی‌شود؛ همان `adId` قبلی دوباره در `callback` برمی‌گردد.
- **دکمهٔ بستن:** هنگام نمایش، دکمهٔ بستن (✕) ابتدا یک **شمارش معکوس ۵ ثانیه‌ای** نشان می‌دهد و سپس فعال می‌شود؛ در این مدت دکمهٔ سخت‌افزاری Back نیز غیرفعال است.
- **انقضای آگهی:** آگهیِ بارگذاری‌شده مدت اعتبار دارد؛ اگر دیر `show` بزنید، `AD_EXPIRED` می‌گیرید و باید دوباره بارگذاری کنید.
- **خطا در نمایش تصویر:** اگر پس از `show` تصویر آگهی رندر نشود، دیالوگ **خودش بسته می‌شود** و خطای `ASSET_LOAD_FAILED` از طریق `onAdFailedToShow` گزارش می‌گردد؛ کاربر با صفحهٔ سیاه یا پیام خطای انگلیسی مواجه نمی‌شود.
- **ارتباط تماماً HTTPS:** درخواست‌های SDK فقط روی `https` انجام می‌شود؛ پکیج هیچ تنظیم cleartext یا `networkSecurityConfig` به برنامهٔ شما اضافه نمی‌کند.
- **چرخهٔ عمر و چرخش صفحه:** SDK به‌صورت خودکار وضعیت دیالوگ آگهی را در تغییرات پیکربندی (مثل چرخش صفحه) مدیریت می‌کند؛ کار اضافه‌ای لازم نیست.
- **مدیریت حافظه:** با فراخوانی `onTrimMemory` از `Application`، کش تصاویر در فشار حافظه آزاد می‌شود.
- **پشتیبانی واقعی از API 21:** جمع‌آوری سیگنال‌های دستگاه روی اندروید ۵ و ۶ نیز مسیر سازگار خود را دارد؛ SDK روی نسخه‌های قدیمی کرش نمی‌کند.

---

## عیب‌یابی مشکلات رایج

<div dir="ltr">

| ⁧نشانه⁩ | ⁧علت و راه‌حل⁩ |
|-------|--------------|
| ⁧`SDK_NOT_INITIALIZED` یا استثنا هنگام `getInterstitialAd`⁩ | ⁧`MagnetAdManager.initialize(...)` را در `Application.onCreate` صدا نزده‌اید یا کلاس Application در مانیفست ثبت نشده است.⁩ |
| ⁧بارگذاری همیشه با `NETWORK_ERROR` شکست می‌خورد⁩ | ⁧دسترسی اینترنت/سرور را بررسی کنید. ارتباط SDK روی `https` است، پس اگر `networkSecurityConfig` سفارشی دارید مطمئن شوید دامنهٔ سرویس را مسدود یا محدود به pinning نکرده باشد.⁩ |
| ⁧`AD_NOT_READY` هنگام `show`⁩ | ⁧یا `adId` نامعتبر/اشتباه است، یا پیش از دریافت `AdLoadResult.Success` صدا زده‌اید، یا همان `adId` قبلاً یک بار مصرف شده است. ابتدا `requestAd` و سپس در کال‌بک آن، با `adId` دریافتی، `show` را صدا بزنید.⁩ |
| ⁧`AD_EXPIRED` هنگام `show`⁩ | ⁧فاصلهٔ بین بارگذاری و نمایش طولانی شده است؛ در کال‌بک خطا دوباره `requestAd` بزنید.⁩ |
| ⁧بارگذاری با `NO_FILL` برمی‌گردد⁩ | ⁧خطای یکپارچه‌سازی نیست؛ سرور در آن لحظه آگهی مناسبی نداشته است. کاربر را با پیام خطا مواجه نکنید و بعداً دوباره تلاش کنید. اگر **همیشه** `NO_FILL` می‌گیرید، درست‌بودن App Id و Placement Id و فعال‌بودن جایگاه در پنل را بررسی کنید.⁩ |
| ⁧بارگذاری با `TIMEOUT` برمی‌گردد⁩ | ⁧کل بارگذاری (درخواست آگهی + دانلود تصویر) در ۱۵ ثانیه تمام نشده است؛ معمولاً به‌دلیل کندی شبکهٔ کاربر. مثل `NO_FILL` با آن برخورد کنید: این بار از نمایش آگهی صرف‌نظر کنید و بعداً دوباره تلاش کنید.⁩ |
| ⁧بارگذاری با `ASSET_LOAD_FAILED` برمی‌گردد⁩ | ⁧دانلود یا دیکود تصویر آگهی ناموفق بوده است — یا شبکه در میانهٔ دانلود قطع شده، یا تصویر از سقف **۱۰ مگابایت** بزرگ‌تر است. اگر برای یک کمپین مشخص تکرار می‌شود، حجم خلاقانه (creative) را به تیم Magnet گزارش دهید.⁩ |
| ⁧آگهی نمایش داده شد ولی بلافاصله بسته شد⁩ | ⁧رندر تصویر شکست خورده و SDK عمداً دیالوگ را بسته است؛ `onAdFailedToShow` با کد `ASSET_LOAD_FAILED` صدا زده می‌شود.⁩ |
| ⁧بازی/برنامه بعد از آگهی ادامه پیدا نمی‌کند⁩ | ⁧ادامهٔ کار را فقط به `onAdDismissed` گره زده‌اید؛ همان منطق را در `onAdFailedToShow` هم بگذارید (بخش [گام ۶](#گام-۶--رویدادها-adlistener-و-نتیجهٔ-بارگذاری-adloadresult)).⁩ |
| ⁧خطای Gradle: `requires compileSdk 34 or later`⁩ | ⁧compileSdk پروژهٔ شما کمتر از ۳۴ است؛ آن را به `34` (یا بالاتر) برسانید.⁩ |
| ⁧مجوز `AD_ID` در برنامهٔ من نباید باشد⁩ | ⁧با `tools:node="remove"` آن را حذف کنید — بخش [دربارهٔ مجوز `AD_ID`](#دربارهٔ-مجوز-ad_id).⁩ |
| ⁧آرتیفکت `com.hasin:magnetad` پیدا نمی‌شود⁩ | ⁧MagnetAd روی هیچ مخزن Maven ای منتشر نمی‌شود؛ خط `implementation("com.hasin:magnetad:…")` را حذف کنید و از روش فایل AAR استفاده کنید — [گام ۱](#گام-۱--افزودن-پکیج-فایل-aar).⁩ |
| ⁧`NoClassDefFoundError` یا `ClassNotFoundException` هنگام اجرا (بیلد سالم بوده)⁩ | ⁧یکی از وابستگی‌های جانبی گام ۱-۲ اضافه نشده است. چون AAR فایل POM ندارد، هیچ‌کدام خودکار آورده نمی‌شوند؛ فهرست را کامل کنید.⁩ |
| ⁧خطای `Duplicate class com.hasin.magnetad...`⁩ | ⁧بیش از یک فایل AAR از MagnetAd در `app/libs/` مانده است؛ نسخه‌های قدیمی را حذف کنید.⁩ |
| ⁧بعد از ارتقای SDK، رفتار قدیمی باقی مانده است⁩ | ⁧فایل AAR جدید را جایگزین کرده‌اید اما نام فایل در `build.gradle.kts` هنوز به نسخهٔ قبلی اشاره می‌کند.⁩ |
| ⁧Gradle نمی‌تواند وابستگی‌ها را دانلود کند⁩ | ⁧`dl.google.com` در دسترس نیست؛ میرور `maven { url = uri("https://maven.myket.ir/") }` را قبل از `google()` قرار دهید.⁩ |
| ⁧خطای `module was compiled with an incompatible version of Kotlin`⁩ | ⁧نسخهٔ Kotlin پروژهٔ شما قدیمی‌تر از 1.9 است؛ آن را به‌روزرسانی کنید.⁩ |
| ⁧کلاس‌های SDK پس از فعال‌کردن Minify حذف می‌شوند⁩ | ⁧معمولاً `consumer-rules.pro` این را پوشش می‌دهد؛ در صورت بروز مشکل، بیلد release را با لاگ بررسی کنید.⁩ |
| ⁧بیلد release با خطای R8 روی `okhttp3`/`okio`/`org.conscrypt` می‌شکند⁩ | ⁧قوانین `-dontwarn` لازم داخل AAR هست؛ مطمئن شوید نسخهٔ به‌روز پکیج را برداشته‌اید. اگر باز هم رخ داد، همان `-dontwarn`ها را موقتاً در `proguard-rules.pro` خودتان بگذارید و به ما اطلاع دهید.⁩ |
| ⁧در بیلد release همهٔ درخواست‌ها `INVALID_RESPONSE` می‌شوند (در debug سالم است)⁩ | ⁧قوانین `kotlinx.serialization` اعمال نشده‌اند — نشانهٔ این است که `consumer-rules.pro` پکیج به بیلد شما نرسیده؛ نسخهٔ AAR/آرتیفکت را بررسی کنید.⁩ |
| ⁧مارکت نصب‌شده باز نمی‌شود و به مرورگر می‌رود⁩ | ⁧بخش `<queries>` هنگام merge حذف شده است؛ خروجی merged manifest بیلد را بررسی کنید (باید سه پکیج مارکت را داشته باشد).⁩ |

</div>

برای دیباگ بهتر، در `AdConfig` مقدار `debugMode = true` قرار دهید تا لاگ‌های SDK با تگ `MagnetAd` در **logcat** چاپ شوند.

---

## محدودیت‌ها

- فقط آگهی **Interstitial** با تصویر (بدون ویدئو/بنر/Rewarded در این نسخه).
- فقط **اندروید** (API 21 به بالا).

---

*نسخهٔ SDK: `1.0.0` — مطابق مقدار `MagnetAdManager.getSDKVersion()`.*

*دریافت پکیج و آخرین نسخه: [github.com/MagnetAds/magad-android-sdk](https://github.com/MagnetAds/magad-android-sdk)*

</div>
