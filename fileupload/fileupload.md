# ch15 파일 업로드/다운로드

## 파일 저장 경로

이 프로젝트는 Eclipse WTP + Tomcat 구조로, 소스 폴더와 실제 런타임 경로가 다릅니다.

| 구분 | 경로 |
|------|------|
| 소스 폴더 (IDE에서 보이는 곳) | `C:\Jsp\myapp\src\main\webapp\ch15\fileupload\` |
| 런타임 배포 경로 (실제 저장 위치) | `C:\Jsp\.metadata\.plugins\org.eclipse.wst.server.core\tmp0\wtpwebapps\myapp\ch15\fileupload\` |

> 업로드된 파일은 **배포 경로**에 저장됩니다. 소스 폴더에 파일이 보이지 않아도 정상입니다.

---

## 관련 파일

| 파일 | 역할 |
|------|------|
| `BoardMgr.java` | 파일 저장 경로 상수 및 업로드/삭제 로직 |
| `BoardPostServlet.java` | 글 작성 요청 처리 → `insertBoard()` 호출 |
| `BoardUpdateServlet.java` | 글 수정 요청 처리 → `MultipartRequest` 생성 + `updateBoard()` 호출 |
| `BoardDeleteServlet.java` | 글 삭제 요청 처리 → `deleteBoard()` 호출 (첨부 파일도 삭제) |
| `post.jsp` | 파일 첨부 포함 글 작성 폼 |
| `update.jsp` | 파일 교체 포함 글 수정 폼 |
| `download.jsp` | 첨부 파일 다운로드 처리 |

---

## 경로 설정 방식

`BoardMgr.java`에 다음과 같이 선언되어 있습니다.

```java
// 웹앱 루트 기준 상대 경로 (컨텍스트 경로)
public static final String SAVEFOLDER = "/ch15/fileupload/";

// 런타임에 실제 절대 경로로 변환
public static String getSaveFolder(HttpServletRequest req) {
    return req.getServletContext().getRealPath(SAVEFOLDER);
}
```

`getRealPath()`는 Tomcat이 배포한 실제 파일시스템 경로를 반환합니다.  
하드코딩된 절대 경로 대신 이 방식을 사용하면 배포 환경이 바뀌어도 경로를 수정할 필요가 없습니다.

---

## 업로드 흐름

```
post.jsp (form enctype="multipart/form-data")
    │
    ▼
BoardPostServlet.doPost(request)
    │
    └─► BoardMgr.insertBoard(req)
            │
            ├─ getSaveFolder(req) → 실제 저장 경로 획득
            ├─ new File(saveFolder).mkdirs() → 폴더 없으면 자동 생성
            ├─ new MultipartRequest(req, saveFolder, ...) → 파일 저장
            └─ INSERT INTO tblBoard (filename, filesize, ...) → DB 기록
```

### 업로드 제한

| 항목 | 값 |
|------|----|
| 최대 파일 크기 | 50 MB (`MAXSIZE = 1202 * 1024 * 50`) |
| 인코딩 | UTF-8 |
| 중복 파일명 처리 | `DefaultFileRenamePolicy` (자동 rename) |

---

## 수정 시 파일 교체 흐름

```
update.jsp (새 파일 첨부)
    │
    ▼
BoardUpdateServlet.doPost(request)
    │
    ├─ getSaveFolder(request) → 경로 획득
    ├─ new MultipartRequest(request, saveFolder, ...) → 새 파일 저장
    │
    └─► BoardMgr.updateBoard(multi, request)
            │
            ├─ 기존 첨부 파일이 있으면 → new File(saveFolder + dbFile).delete()
            └─ UPDATE tblBoard (filename, filesize) → DB 갱신
```

---

## 삭제 시 파일 삭제 흐름

```
read.jsp (삭제 요청)
    │
    ▼
BoardDeleteServlet.doPost(request)
    │
    └─► BoardMgr.deleteBoard(num, request)
            │
            ├─ DB에서 filename 조회
            ├─ new File(getSaveFolder(req) + filename).delete() → 파일 삭제
            └─ DELETE FROM tblBoard WHERE num = ? → DB 레코드 삭제
```

---

## 다운로드 흐름

```
read.jsp → download.jsp?filename=xxx
    │
    ▼
download.jsp
    ├─ BoardMgr.getSaveFolder(request) → 실제 경로 획득
    ├─ new File(saveFolder, filename) → 파일 객체 생성
    ├─ Content-Disposition 헤더 설정 (브라우저별 인코딩 분기)
    └─ BufferedInputStream / BufferedOutputStream → 파일 스트림 전송
```

### 브라우저별 파일명 인코딩

| 브라우저 | 인코딩 처리 |
|----------|------------|
| IE / Trident | `EUC-KR → ISO-8859-1` 변환 |
| 그 외 (Chrome, Firefox 등) | `UTF-8 → ISO-8859-1` 변환 |

---

## 주의사항

- **배포 경로와 소스 경로는 다릅니다.** 업로드된 파일을 확인할 때는 `tmp0/wtpwebapps/myapp/ch15/fileupload/`를 확인하세요.
- Eclipse에서 **Publish**(재배포)를 실행하면 배포 폴더가 갱신됩니다. 이때 배포 폴더에만 있는 업로드 파일은 삭제되지 않지만, 운영 환경에서는 별도의 외부 스토리지 경로 사용을 권장합니다.
- `SAVEFOLDER` 상수는 경로 문자열로만 사용하고, 실제 파일 I/O에는 항상 `getSaveFolder(request)`를 통해 절대 경로를 얻어야 합니다.
