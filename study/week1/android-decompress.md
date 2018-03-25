# android-decompress
> java.util 패키지의 클래스들을 이용해 안드로이드에서 파일 압축 해제하기

> present time : 2018-03-25-SUN  
> last updated : 2018-03-26-01:19-seohyun99

----------------

## 0. 사용하는 클래스
* [ZipEntry](https://developer.android.com/reference/java/util/zip/ZipEntry.html)
* [ZipInputStream](https://developer.android.com/reference/java/util/zip/ZipInputStream.html)
* [FileOutputStream](https://developer.android.com/reference/java/io/FileOutputStream.html)
* [FileInputStream](https://developer.android.com/reference/java/io/FileInputStream.html)

### 0-1. 클래스 간단 요약

**1. ZipEntry**
> This class is used to represent a ZIP file entry.  
>  
> 이 클래스는 ZIP 파일 항목을 나타내는 데 사용됩니다.

**2. ZipInputStream**
> This class implements an input stream filter for reading files in the ZIP file format. Includes support for both compressed and uncompressed entries.  
>
> 이 클래스는 ZIP 파일 형식의 파일을 읽기위한 input stream filter를 구현합니다. 압축된 항목과 압축되지 않은 항목을 모두 지원합니다.

**3. FileInputStream**
> A FileInputStream obtains input bytes from a file in a file system. What files are available depends on the host environment.  
>
> FileInputStream은 파일 시스템의 파일로부터 입력 바이트를 획득합니다. 사용할 수있는 파일은 호스트 환경에 따라 다릅니다.


----------------

### 1. 권한 & 해제 위치 관련

: 외부 저장소에 파일 압축 해제를 하기 위해서는 아래와 같은 권한이 필요합니다.  
**< uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />**

[관련 링크](https://developer.android.com/training/basics/data-storage/files.html?hl=ko)

: 내부 저장소에 압축 해제의 경우 'cotnext.getFilesDir()' 을 통해 앱에 대한 내부 디렉토리를 가져올 수 있습니다.


----------------

### 2. 압축 해제 기본 소스

    private static final int BUFFER_SIZE = 1024 * 2;

    /**
    * Zip 파일의 압축을 푼다.
    *
    * @param zipFile - 압축 풀 Zip 파일
    * @throws Throwable
    */*
    public void decompress(File zipFile) throws Throwable {
        ZipEntry zipentry = null;
        ZipInputStream zis = null;
        FileInputStream fis = null;

        try {
            fis = new FileInputStream(zipFile);
            zis = new ZipInputStream(fis);

            // entry가 없을때까지 뽑기
            while (zis.getNextEntry() != null) {
                ZipEntry zipentry = zis.getNextEntry();
                String filename = zipentry.getName();

                File file = new File(context.getFilesDir(), fileName);

                if (zipentry.isDirectory()) {
                    file.mkdirs();
                } else {
                    try {
                        createFile(file, zis);
                    } catch (Throwable th) {
                        Log.e("E", "E", th);
                    }
                }
            }
        } catch (Throwable e) {
            throw e;
        } finally {
            if (fis != null) {
                fis.close();
            }
            if (fos != null) {
                fos.close();
            }
            if (zis != null) {
                zis.close();
            }
        }
    }

    /**
     * ZipEntry의 압축을 푼다.
     *
     * @param file - 압축 해제 될 파일
     * @param zis - ZipInputStream
     * @throws Throwable
     */*
    private void createFile(File file, ZipInputStream zis) throws Throwable {
        FileOutputStream fos = null;
        try {
            fos = new FileOutputStream(file);
            byte[] buffer = new byte[BUFFER_SIZE];
            int size = 0;

            while ((size = zis.read(buffer)) > 0) {
                // byte로 파일 만들기
                fos.write(buffer, 0, size);
            }
        } catch (Throwable e) {
            throw e;
        } finally {
            if (fos != null) {
                fos.close();
            }
        }
    }

----------------

### 3. 예외 상황
압축 해제하는 파일에 한글 폴더나 파일을 가지고 있는 경우 인코딩 문제로 한글이 깨지게 됩니다.  
아래 코드는 ZipInputStream 생성자 코드인데, charset이 UTF_8로 세팅되어 있는 모습을 볼 수 있습니다.

    /**
    * Creates a new ZIP input stream.
    *
    * <p>The UTF-8 {@link java.nio.charset.Charset charset} is used to
    * decode the entry names.
    *
    * @param in the actual input stream
    */*
    public ZipInputStream(InputStream in) {
        this(in, StandardCharsets.UTF_8);
    }


해당 현상을 없애기 위해서는 [jazzlib](http://jazzlib.sourceforge.net) 의 라이브러리를 통해 해결 할 수 있습니다.  
코드의 수정 없이, import문을 해당 라이브러리로 수정하기만 하면 해결됩니다.
