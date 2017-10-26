
第一步：配置app下的Gradle文件

apply plugin: 'com.android.application'

dependencies {
compile fileTree(include: ['*.jar'], dir: 'libs')
androidTestCompile('com.android.support.test.espresso:espresso-core:2.2.2', {
exclude group: 'com.android.support', module: 'support-annotations'
})
compile 'com.android.support:appcompat-v7:25.3.1'
compile 'com.android.support:design:25.3.1'
compile 'com.android.support.constraint:constraint-layout:1.0.2'
testCompile 'junit:junit:4.12'
compile("com.tencent.tinker:tinker-android-lib:${TINKER_VERSION}") { changing = true }
provided("com.tencent.tinker:tinker-android-anno:${TINKER_VERSION}") { changing = true }
compile 'com.android.support:multidex:1.0.1'
//use to test multiDex
// compile group: 'com.google.guava', name: 'guava', version: '19.0'
// compile "org.scala-lang:scala-library:2.11.7"
//use for local maven test
// compile("com.tencent.tinker:tinker-android-loader:${TINKER_VERSION}") { changing = true }
// compile("com.tencent.tinker:aosp-dexutils:${TINKER_VERSION}") { changing = true }
// compile("com.tencent.tinker:bsdiff-util:${TINKER_VERSION}") { changing = true }
// compile("com.tencent.tinker:tinker-commons:${TINKER_VERSION}") { changing = true }
}

def gitSha() {
try {
String gitRev = 'v1.9.5'
if (gitRev == null) {
throw new GradleException("can't get git rev, you should add git to system path or just input test value, such as 'testTinkerId'")
}
return gitRev
} catch (Exception e) {
throw new GradleException("can't get git rev, you should add git to system path or just input test value, such as 'testTinkerId'")
}
}

def javaVersion = JavaVersion.VERSION_1_7

android {
compileSdkVersion 25
buildToolsVersion "25.0.2"

compileOptions {
sourceCompatibility javaVersion
targetCompatibility javaVersion
}
//recommend
dexOptions {
jumboMode = true
}

signingConfigs {
release {
try {
storeFile file("./keystore/release.keystore")
storePassword "testres"
keyAlias "testres"
keyPassword "testres"
} catch (ex) {
throw new InvalidUserDataException(ex.toString())
}
}
debug {
}
}

defaultConfig {
applicationId "tinker.sample.android"
minSdkVersion 14
targetSdkVersion 22
versionCode 1
versionName "1.0.0"
/**
* you can use multiDex and install it in your ApplicationLifeCycle implement
*/
multiDexEnabled true
/**
* buildConfig can change during patch!
* we can use the newly value when patch
*/
buildConfigField "String", "MESSAGE", "\"I am the base apk\""
// buildConfigField "String", "MESSAGE", "\"I am the patch apk\""
/**
* client version would update with patch
* so we can get the newly git version easily!
*/
buildConfigField "String", "TINKER_ID", "\"${getTinkerIdValue()}\""
buildConfigField "String", "PLATFORM", "\"all\""
}

// aaptOptions{
// cruncherEnabled false
// }

// //use to test flavors support
// productFlavors {
// flavor1 {
// applicationId 'tinker.sample.android.flavor1'
// }
//
// flavor2 {
// applicationId 'tinker.sample.android.flavor2'
// }
// }

buildTypes {
release {
minifyEnabled true
signingConfig signingConfigs.release
proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
}
debug {
debuggable true
minifyEnabled false
signingConfig signingConfigs.debug
}
}
sourceSets {
main {
jniLibs.srcDirs = ['libs']
}
}
}

def bakPath = file("${buildDir}/bakApk/")

/**
* you can use assembleRelease to build you base apk
* use tinkerPatchRelease -POLD_APK= -PAPPLY_MAPPING= -PAPPLY_RESOURCE= to build patch
* add apk from the build/bakApk
*/
ext {
//for some reason, you may want to ignore tinkerBuild, such as instant run debug build?
tinkerEnabled = true

//for normal build
//old apk file to build patch apk
tinkerOldApkPath = "${bakPath}/app-debug-0811-16-37-28.apk"
//proguard mapping file to build patch apk
tinkerApplyMappingPath = "${bakPath}/app-debug-1018-17-32-47-mapping.txt"
//resource R.txt to build patch apk, must input if there is resource changed
tinkerApplyResourcePath = "${bakPath}/app-debug-0811-16-37-28-R.txt"

//only use for build all flavor, if not, just ignore this field
tinkerBuildFlavorDirectory = "${bakPath}/app-1018-17-32-47"
}


def getOldApkPath() {
return hasProperty("OLD_APK") ? OLD_APK : ext.tinkerOldApkPath
}

def getApplyMappingPath() {
return hasProperty("APPLY_MAPPING") ? APPLY_MAPPING : ext.tinkerApplyMappingPath
}

def getApplyResourceMappingPath() {
return hasProperty("APPLY_RESOURCE") ? APPLY_RESOURCE : ext.tinkerApplyResourcePath
}

def getTinkerIdValue() {
return hasProperty("TINKER_ID") ? TINKER_ID : gitSha()
}

def buildWithTinker() {
return hasProperty("TINKER_ENABLE") ? TINKER_ENABLE : ext.tinkerEnabled
}

def getTinkerBuildFlavorDirectory() {
return ext.tinkerBuildFlavorDirectory
}

if (buildWithTinker()) {
apply plugin: 'com.tencent.tinker.patch'

tinkerPatch {
/**
* necessary，default 'null'
* the old apk path, use to diff with the new apk to build
* add apk from the build/bakApk
*/
oldApk = getOldApkPath()
/**
* optional，default 'false'
* there are some cases we may get some warnings
* if ignoreWarning is true, we would just assert the patch process
* case 1: minSdkVersion is below 14, but you are using dexMode with raw.
* it must be crash when load.
* case 2: newly added Android Component in AndroidManifest.xml,
* it must be crash when load.
* case 3: loader classes in dex.loader{} are not keep in the main dex,
* it must be let tinker not work.
* case 4: loader classes in dex.loader{} changes,
* loader classes is ues to load patch dex. it is useless to change them.
* it won't crash, but these changes can't effect. you may ignore it
* case 5: resources.arsc has changed, but we don't use applyResourceMapping to build
*/
ignoreWarning = false

/**
* optional，default 'true'
* whether sign the patch file
* if not, you must do yourself. otherwise it can't check success during the patch loading
* we will use the sign config with your build type
*/
useSign = true

/**
* optional，default 'true'
* whether use tinker to build
*/
tinkerEnable = buildWithTinker()

/**
* Warning, applyMapping will affect the normal android build!
*/
buildConfig {
/**
* optional，default 'null'
* if we use tinkerPatch to build the patch apk, you'd better to apply the old
* apk mapping file if minifyEnabled is enable!
* Warning:
* you must be careful that it will affect the normal assemble build!
*/
applyMapping = getApplyMappingPath()
/**
* optional，default 'null'
* It is nice to keep the resource id from R.txt file to reduce java changes
*/
applyResourceMapping = getApplyResourceMappingPath()

/**
* necessary，default 'null'
* because we don't want to check the base apk with md5 in the runtime(it is slow)
* tinkerId is use to identify the unique base apk when the patch is tried to apply.
* we can use git rev, svn rev or simply versionCode.
* we will gen the tinkerId in your manifest automatic
*/
tinkerId = getTinkerIdValue()

/**
* if keepDexApply is true, class in which dex refer to the old apk.
* open this can reduce the dex diff file size.
*/
keepDexApply = false

/**
* optional, default 'false'
* Whether tinker should treat the base apk as the one being protected by app
* protection tools.
* If this attribute is true, the generated patch package will contain a
* dex including all changed classes instead of any dexdiff patch-info files.
*/
isProtectedApp = false
}

dex {
/**
* optional，default 'jar'
* only can be 'raw' or 'jar'. for raw, we would keep its original format
* for jar, we would repack dexes with zip format.
* if you want to support below 14, you must use jar
* or you want to save rom or check quicker, you can use raw mode also
*/
dexMode = "jar"

/**
* necessary，default '[]'
* what dexes in apk are expected to deal with tinkerPatch
* it support * or ? pattern.
*/
pattern = ["classes*.dex",
"assets/secondary-dex-?.jar"]
/**
* necessary，default '[]'
* Warning, it is very very important, loader classes can't change with patch.
* thus, they will be removed from patch dexes.
* you must put the following class into main dex.
* Simply, you should add your own application {@code tinker.sample.android.SampleApplication}
* own tinkerLoader, and the classes you use in them
*
*/
loader = [
//use sample, let BaseBuildInfo unchangeable with tinker
"tinker.sample.android.app.BaseBuildInfo"
]
}

lib {
/**
* optional，default '[]'
* what library in apk are expected to deal with tinkerPatch
* it support * or ? pattern.
* for library in assets, we would just recover them in the patch directory
* you can get them in TinkerLoadResult with Tinker
*/
pattern = ["lib/*/*.so"]
}

res {
/**
* optional，default '[]'
* what resource in apk are expected to deal with tinkerPatch
* it support * or ? pattern.
* you must include all your resources in apk here,
* otherwise, they won't repack in the new apk resources.
*/
pattern = ["res/*", "assets/*", "resources.arsc", "AndroidManifest.xml"]

/**
* optional，default '[]'
* the resource file exclude patterns, ignore add, delete or modify resource change
* it support * or ? pattern.
* Warning, we can only use for files no relative with resources.arsc
*/
ignoreChange = ["assets/sample_meta.txt"]

/**
* default 100kb
* for modify resource, if it is larger than 'largeModSize'
* we would like to use bsdiff algorithm to reduce patch file size
*/
largeModSize = 100
}

packageConfig {
/**
* optional，default 'TINKER_ID, TINKER_ID_VALUE' 'NEW_TINKER_ID, NEW_TINKER_ID_VALUE'
* package meta file gen. path is assets/package_meta.txt in patch file
* you can use securityCheck.getPackageProperties() in your ownPackageCheck method
* or TinkerLoadResult.getPackageConfigByName
* we will get the TINKER_ID from the old apk manifest for you automatic,
* other config files (such as patchMessage below)is not necessary
*/
configField("patchMessage", "tinker is sample to use")
/**
* just a sample case, you can use such as sdkVersion, brand, channel...
* you can parse it in the SamplePatchListener.
* Then you can use patch conditional!
*/
configField("platform", "all")
/**
* patch version via packageConfig
*/
configField("patchVersion", "1.0")
}
//or you can add config filed outside, or get meta value from old apk
//project.tinkerPatch.packageConfig.configField("test1", project.tinkerPatch.packageConfig.getMetaDataFromOldApk("Test"))
//project.tinkerPatch.packageConfig.configField("test2", "sample")

/**
* if you don't use zipArtifact or path, we just use 7za to try
*/
sevenZip {
/**
* optional，default '7za'
* the 7zip artifact path, it will use the right 7za with your platform
*/
zipArtifact = "com.tencent.mm:SevenZip:1.1.10"
/**
* optional，default '7za'
* you can specify the 7za path yourself, it will overwrite the zipArtifact value
*/
// path = "/usr/local/bin/7za"
}
}

List<String> flavors = new ArrayList<>();
project.android.productFlavors.each {flavor ->
flavors.add(flavor.name)
}
boolean hasFlavors = flavors.size() > 0
def date = new Date().format("MMdd-HH-mm-ss")

/**
* bak apk and mapping
*/
android.applicationVariants.all { variant ->
/**
* task type, you want to bak
*/
def taskName = variant.name

tasks.all {
if ("assemble${taskName.capitalize()}".equalsIgnoreCase(it.name)) {

it.doLast {
copy {
def fileNamePrefix = "${project.name}-${variant.baseName}"
def newFileNamePrefix = hasFlavors ? "${fileNamePrefix}" : "${fileNamePrefix}-${date}"

def destPath = hasFlavors ? file("${bakPath}/${project.name}-${date}/${variant.flavorName}") : bakPath
from variant.outputs.outputFile
into destPath
rename { String fileName ->
fileName.replace("${fileNamePrefix}.apk", "${newFileNamePrefix}.apk")
}

from "${buildDir}/outputs/mapping/${variant.dirName}/mapping.txt"
into destPath
rename { String fileName ->
fileName.replace("mapping.txt", "${newFileNamePrefix}-mapping.txt")
}

from "${buildDir}/intermediates/symbols/${variant.dirName}/R.txt"
into destPath
rename { String fileName ->
fileName.replace("R.txt", "${newFileNamePrefix}-R.txt")
}
}
}
}
}
}
project.afterEvaluate {
//sample use for build all flavor for one time
if (hasFlavors) {
task(tinkerPatchAllFlavorRelease) {
group = 'tinker'
def originOldPath = getTinkerBuildFlavorDirectory()
for (String flavor : flavors) {
def tinkerTask = tasks.getByName("tinkerPatch${flavor.capitalize()}Release")
dependsOn tinkerTask
def preAssembleTask = tasks.getByName("process${flavor.capitalize()}ReleaseManifest")
preAssembleTask.doFirst {
String flavorName = preAssembleTask.name.substring(7, 8).toLowerCase() + preAssembleTask.name.substring(8, preAssembleTask.name.length() - 15)
project.tinkerPatch.oldApk = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-release.apk"
project.tinkerPatch.buildConfig.applyMapping = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-release-mapping.txt"
project.tinkerPatch.buildConfig.applyResourceMapping = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-release-R.txt"

}

}
}

task(tinkerPatchAllFlavorDebug) {
group = 'tinker'
def originOldPath = getTinkerBuildFlavorDirectory()
for (String flavor : flavors) {
def tinkerTask = tasks.getByName("tinkerPatch${flavor.capitalize()}Debug")
dependsOn tinkerTask
def preAssembleTask = tasks.getByName("process${flavor.capitalize()}DebugManifest")
preAssembleTask.doFirst {
String flavorName = preAssembleTask.name.substring(7, 8).toLowerCase() + preAssembleTask.name.substring(8, preAssembleTask.name.length() - 13)
project.tinkerPatch.oldApk = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-debug.apk"
project.tinkerPatch.buildConfig.applyMapping = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-debug-mapping.txt"
project.tinkerPatch.buildConfig.applyResourceMapping = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-debug-R.txt"
}

}
}
}
}
}

第二步：配置Application新建SimpleTinkerInApplicationLike文件

@DefaultLifeCycle(application = ".SimpleTinkerInApplication",
flags = ShareConstants.TINKER_ENABLE_ALL,
loadVerifyFlag = false)
public class SimpleTinkerInApplicationLike extends ApplicationLike {
public SimpleTinkerInApplicationLike(Application application, int tinkerFlags, boolean tinkerLoadVerifyFlag, long applicationStartElapsedTime, long applicationStartMillisTime, Intent tinkerResultIntent) {
super(application, tinkerFlags, tinkerLoadVerifyFlag, applicationStartElapsedTime, applicationStartMillisTime, tinkerResultIntent);
}

@Override
public void onBaseContextAttached(Context base) {
super.onBaseContextAttached(base);
}

@Override
public void onCreate() {
super.onCreate();
TinkerInstaller.install(this);
}
}


然后在配置文件中配置

<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
<application
android:name=".SimpleTinkerInApplication"
android:allowBackup="true"
android:icon="@mipmap/ic_launcher"
android:label="@string/app_name"
android:roundIcon="@mipmap/ic_launcher_round"
android:supportsRtl="true"
android:theme="@style/AppTheme">

在gradle.properties文件中定义变量

TINKER_VERSION=1.8.0

在项目目录下的build.gradle文件中配置依赖

buildscript {
repositories {
mavenLocal()
jcenter()
}
dependencies {
classpath 'com.android.tools.build:gradle:2.3.3'
//tinker依懒
classpath "com.tencent.tinker:tinker-patch-gradle-plugin:${TINKER_VERSION}"
// NOTE: Do not place your application dependencies here; they belong
// in the individual module build.gradle files
}
}

allprojects {
repositories {
mavenLocal()
jcenter()
}
}

task clean(type: Delete) {
delete rootProject.buildDir
}


在配置过程中可能会遇到很多问题比如：

Error:Execution failed for task ':app:validateDebugSigning'. > Keystore file F:\myAndroid3\android_s

http://blog.csdn.net/yzwty/article/details/64603276


can't get git rev, you should add git to system path or just input test value, such as 'testTinkerId

http://blog.csdn.net/gyh790005156/article/details/52796641
除了这个链接可以尝试写死git版本号，比如在gradle文件中修改



还有就是在生成补丁的时候点击studio右侧gradle选择tinker下面的tinkerPathDebug双击生成，在outputs目录下tinkerPath下的path_signed_7zip.apk，这个就是补丁，
降这个补丁放在app存储根目录下，再次运行app的时候就会加载这个补丁，从而实现热修复的效果，但是在这个补丁的位置是程序中指定的，除了存储权限之外，还有必须要注意下面：





