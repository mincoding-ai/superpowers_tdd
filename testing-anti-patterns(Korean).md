# 테스팅 안티패턴

**이 문서를 로드하는 경우:** 테스트를 작성하거나 변경할 때, mock을 추가할 때, 또는 프로덕션 코드에 테스트 전용 메서드를 추가하고 싶을 때.

## 개요

테스트는 mock의 동작이 아닌 실제 동작을 검증해야 한다. Mock은 격리를 위한 수단이지, 테스트 대상이 아니다.

**핵심 원칙:** 코드가 무엇을 하는지를 테스트하라, mock이 무엇을 하는지가 아니라.

**엄격한 TDD를 따르면 이런 안티패턴을 예방할 수 있다.**

## 철칙

```
1. mock 동작을 절대 테스트하지 마라
2. 프로덕션 클래스에 테스트 전용 메서드를 절대 추가하지 마라
3. 의존성을 이해하지 않고 절대 mock하지 마라
```

## 안티패턴 1: Mock 동작 테스트

**위반 사례:**

**Java**
```java
// ❌ 나쁨: mock이 호출됐는지를 테스트, 실제 동작이 아님
@Test
void renders_sidebar() {
    Sidebar mockSidebar = mock(Sidebar.class);
    when(mockSidebar.render()).thenReturn("<div data-testid='sidebar-mock'/>");
    Page page = new Page(mockSidebar);
    page.render();
    verify(mockSidebar).render();
}
```

**C++**
```cpp
// ❌ 나쁨: mock이 호출됐는지를 테스트, 실제 동작이 아님
TEST(PageTest, RendersSidebar) {
    MockSidebar mock_sidebar;
    EXPECT_CALL(mock_sidebar, render())
        .WillOnce(Return("<div data-testid='sidebar-mock'/>"));
    Page page(&mock_sidebar);
    page.render();
}
```

**Python**
```python
# ❌ 나쁨: mock이 호출됐는지를 테스트, 실제 동작이 아님
def test_renders_sidebar(mocker):
    mock_sidebar = mocker.Mock()
    mock_sidebar.render.return_value = "<div data-testid='sidebar-mock'/>"
    page = Page(mock_sidebar)
    page.render()
    mock_sidebar.render.assert_called_once()
```

**왜 잘못됐는가:**
- 컴포넌트가 동작하는지가 아니라 mock이 동작하는지를 검증하고 있다
- mock이 있으면 통과, 없으면 실패
- 실제 동작에 대해 아무것도 알려주지 않는다

**담당자의 지적:** "mock의 동작을 테스트하고 있는 건 아닌가요?"

**해결책:**

**Java**
```java
// ✅ 좋음: 실제 컴포넌트를 테스트하거나 mock을 사용하지 마라
@Test
void renders_sidebar() {
    Page page = new Page(new RealSidebar()); // sidebar를 mock하지 않음
    String output = page.render();
    assertThat(output).contains("<nav");
}
```

**C++**
```cpp
// ✅ 좋음: 실제 컴포넌트를 테스트하거나 mock을 사용하지 마라
TEST(PageTest, RendersSidebar) {
    Page page(std::make_unique<RealSidebar>()); // sidebar를 mock하지 않음
    std::string output = page.render();
    EXPECT_THAT(output, HasSubstr("<nav"));
}
```

**Python**
```python
# ✅ 좋음: 실제 컴포넌트를 테스트하거나 mock을 사용하지 마라
def test_renders_sidebar():
    page = Page(RealSidebar())  # sidebar를 mock하지 않음
    output = page.render()
    assert "<nav" in output
```

격리를 위해 반드시 mock해야 한다면: mock에 대해 단언하지 말고, sidebar가 있을 때 Page의 동작을 테스트하라.

### 게이트 함수

```
mock 요소에 대해 단언하기 전에:
  질문: "실제 컴포넌트 동작을 테스트하는가, mock의 존재만 테스트하는가?"

  mock의 존재를 테스트하는 경우:
    멈춰라 - 단언을 삭제하거나 컴포넌트의 mock을 제거하라

  대신 실제 동작을 테스트하라
```

## 안티패턴 2: 프로덕션 코드의 테스트 전용 메서드

**위반 사례:**

**Java**
```java
// ❌ 나쁨: destroy()는 테스트에서만 사용됨
class Session {
    public void destroy() { // 프로덕션 API처럼 보임!
        workspaceManager.destroyWorkspace(this.id);
        // ... 정리 작업
    }
}

// 테스트에서
@AfterEach
void tearDown() { session.destroy(); }
```

**C++**
```cpp
// ❌ 나쁨: destroy()는 테스트에서만 사용됨
class Session {
public:
    void destroy() { // 프로덕션 API처럼 보임!
        workspace_manager_->destroyWorkspace(id_);
        // ... 정리 작업
    }
};

// 테스트에서
void TearDown() override { session_->destroy(); }
```

**Python**
```python
# ❌ 나쁨: destroy()는 테스트에서만 사용됨
class Session:
    def destroy(self):  # 프로덕션 API처럼 보임!
        self.workspace_manager.destroy_workspace(self.id)
        # ... 정리 작업

# 테스트에서
def teardown_method(self):
    self.session.destroy()
```

**왜 잘못됐는가:**
- 프로덕션 클래스가 테스트 전용 코드로 오염됨
- 프로덕션에서 실수로 호출될 위험이 있음
- YAGNI와 관심사 분리 원칙 위반
- 객체 생명주기와 엔티티 생명주기를 혼동함

**해결책:**

**Java**
```java
// ✅ 좋음: 테스트 유틸리티가 테스트 정리를 담당
// Session은 destroy()가 없음 - 프로덕션에서는 상태가 없음

// test-utils/에서
public class SessionTestUtils {
    public static void cleanupSession(Session session, WorkspaceManager workspaceManager) {
        WorkspaceInfo workspace = session.getWorkspaceInfo();
        if (workspace != null) {
            workspaceManager.destroyWorkspace(workspace.getId());
        }
    }
}

// 테스트에서
@AfterEach
void tearDown() { SessionTestUtils.cleanupSession(session, workspaceManager); }
```

**C++**
```cpp
// ✅ 좋음: 테스트 유틸리티가 테스트 정리를 담당
// Session은 destroy()가 없음 - 프로덕션에서는 상태가 없음

// test_utils/session_test_utils.h에서
void cleanupSession(Session& session, WorkspaceManager& manager) {
    auto workspace = session.getWorkspaceInfo();
    if (workspace) {
        manager.destroyWorkspace(workspace->id);
    }
}

// 테스트에서
void TearDown() override { cleanupSession(*session_, *workspace_manager_); }
```

**Python**
```python
# ✅ 좋음: 테스트 유틸리티가 테스트 정리를 담당
# Session은 destroy()가 없음 - 프로덕션에서는 상태가 없음

# test_utils/session_test_utils.py에서
def cleanup_session(session, workspace_manager):
    workspace = session.get_workspace_info()
    if workspace:
        workspace_manager.destroy_workspace(workspace.id)

# 테스트에서
def teardown_method(self):
    cleanup_session(self.session, self.workspace_manager)
```

### 게이트 함수

```
프로덕션 클래스에 메서드를 추가하기 전에:
  질문: "이 메서드는 테스트에서만 사용되는가?"

  그렇다면:
    멈춰라 - 추가하지 마라
    테스트 유틸리티에 넣어라

  질문: "이 클래스가 이 리소스의 생명주기를 소유하는가?"

  아니라면:
    멈춰라 - 이 메서드에 맞는 클래스가 아니다
```

## 안티패턴 3: 이해 없는 Mock 사용

**위반 사례:**

**Java**
```java
// ❌ 나쁨: mock이 테스트 로직을 망가뜨린다
@Test
void detects_duplicate_server() throws Exception {
    // Mock이 테스트가 의존하는 설정 파일 쓰기를 막는다!
    ToolCatalog mockCatalog = mock(ToolCatalog.class);
    when(mockCatalog.discoverAndCacheTools(any())).thenReturn(null);

    addServer(config);
    addServer(config); // 예외가 발생해야 하지만, 발생하지 않는다!
}
```

**C++**
```cpp
// ❌ 나쁨: mock이 테스트 로직을 망가뜨린다
TEST(ServerTest, DetectsDuplicateServer) {
    // Mock이 테스트가 의존하는 설정 파일 쓰기를 막는다!
    MockToolCatalog mock_catalog;
    EXPECT_CALL(mock_catalog, discoverAndCacheTools(_))
        .WillRepeatedly(Return());

    addServer(config);
    addServer(config); // 예외가 발생해야 하지만, 발생하지 않는다!
}
```

**Python**
```python
# ❌ 나쁨: mock이 테스트 로직을 망가뜨린다
def test_detects_duplicate_server(mocker):
    # Mock이 테스트가 의존하는 설정 파일 쓰기를 막는다!
    mocker.patch.object(ToolCatalog, 'discover_and_cache_tools', return_value=None)

    add_server(config)
    add_server(config)  # 예외가 발생해야 하지만, 발생하지 않는다!
```

**왜 잘못됐는가:**
- mock된 메서드가 테스트가 의존하는 부수 효과(설정 파일 쓰기)를 가지고 있었다
- "안전을 위해" 과도하게 mock하면 실제 동작이 망가진다
- 잘못된 이유로 통과하거나 이유 없이 실패한다

**해결책:**

**Java**
```java
// ✅ 좋음: 올바른 수준에서 mock
@Test
void detects_duplicate_server() throws Exception {
    // 느린 부분만 mock하고, 테스트에 필요한 동작은 보존
    MCPServerManager mockManager = mock(MCPServerManager.class); // 느린 서버 시작만 mock

    addServer(config);  // 설정 파일 기록됨
    addServer(config);  // 중복 감지 ✓
}
```

**C++**
```cpp
// ✅ 좋음: 올바른 수준에서 mock
TEST(ServerTest, DetectsDuplicateServer) {
    // 느린 부분만 mock하고, 테스트에 필요한 동작은 보존
    MockMCPServerManager mock_manager; // 느린 서버 시작만 mock

    addServer(config);  // 설정 파일 기록됨
    addServer(config);  // 중복 감지 ✓
}
```

**Python**
```python
# ✅ 좋음: 올바른 수준에서 mock
def test_detects_duplicate_server(mocker):
    # 느린 부분만 mock하고, 테스트에 필요한 동작은 보존
    mocker.patch.object(MCPServerManager, 'start')  # 느린 서버 시작만 mock

    add_server(config)  # 설정 파일 기록됨
    add_server(config)  # 중복 감지 ✓
```

### 게이트 함수

```
메서드를 mock하기 전에:
  멈춰라 - 아직 mock하지 마라

  1. 질문: "실제 메서드의 부수 효과는 무엇인가?"
  2. 질문: "이 테스트가 그 부수 효과 중 어느 것에 의존하는가?"
  3. 질문: "이 테스트에 무엇이 필요한지 완전히 이해하고 있는가?"

  부수 효과에 의존하는 경우:
    더 낮은 수준(실제 느리거나 외부인 작업)에서 mock하라
    또는 필요한 동작을 보존하는 테스트 더블을 사용하라
    테스트가 의존하는 고수준 메서드를 mock하지 마라

  테스트가 무엇에 의존하는지 불확실한 경우:
    실제 구현으로 먼저 테스트를 실행하라
    실제로 무슨 일이 일어나야 하는지 관찰하라
    그런 다음 올바른 수준에서 최소한의 mock을 추가하라

  레드 플래그:
    - "안전을 위해 이걸 mock할게"
    - "이게 느릴 수 있으니까 mock하는 게 낫겠어"
    - 의존성 체인을 이해하지 않고 mock 사용
```

## 안티패턴 4: 불완전한 Mock

**위반 사례:**

**Java**
```java
// ❌ 나쁨: 부분 mock - 필요하다고 생각하는 필드만
ApiResponse mockResponse = new ApiResponse();
mockResponse.setStatus("success");
mockResponse.setData(new UserData("123", "Alice"));
// 누락: 하위 코드가 사용하는 metadata

// 나중에: response.getMetadata().getRequestId()에 접근할 때 깨진다
```

**C++**
```cpp
// ❌ 나쁨: 부분 mock - 필요하다고 생각하는 필드만
ApiResponse mock_response;
mock_response.status = "success";
mock_response.data = UserData{"123", "Alice"};
// 누락: 하위 코드가 사용하는 metadata

// 나중에: mock_response.metadata.request_id에 접근할 때 크래시
```

**Python**
```python
# ❌ 나쁨: 부분 mock - 필요하다고 생각하는 필드만
mock_response = ApiResponse(
    status="success",
    data=UserData(user_id="123", name="Alice"),
    # 누락: 하위 코드가 사용하는 metadata
)

# 나중에: response.metadata.request_id에 접근할 때 AttributeError
```

**왜 잘못됐는가:**
- **부분 mock은 구조적 가정을 숨긴다** - 알고 있는 필드만 mock했다
- **하위 코드가 포함하지 않은 필드에 의존할 수 있다** - 조용한 실패
- **테스트는 통과하지만 통합은 실패한다** - mock은 불완전, 실제 API는 완전
- **거짓된 확신** - 테스트가 실제 동작에 대해 아무것도 증명하지 않는다

**철칙:** 즉각적인 테스트에서 사용하는 필드만이 아닌, 실제로 존재하는 완전한 데이터 구조를 mock하라.

**해결책:**

**Java**
```java
// ✅ 좋음: 실제 API의 완전성을 반영
ApiResponse mockResponse = new ApiResponse();
mockResponse.setStatus("success");
mockResponse.setData(new UserData("123", "Alice"));
mockResponse.setMetadata(new Metadata("req-789", 1234567890L));
// 실제 API가 반환하는 모든 필드
```

**C++**
```cpp
// ✅ 좋음: 실제 API의 완전성을 반영
ApiResponse mock_response;
mock_response.status = "success";
mock_response.data = UserData{"123", "Alice"};
mock_response.metadata = Metadata{"req-789", 1234567890};
// 실제 API가 반환하는 모든 필드
```

**Python**
```python
# ✅ 좋음: 실제 API의 완전성을 반영
mock_response = ApiResponse(
    status="success",
    data=UserData(user_id="123", name="Alice"),
    metadata=Metadata(request_id="req-789", timestamp=1234567890),
    # 실제 API가 반환하는 모든 필드
)
```

### 게이트 함수

```
mock 응답을 생성하기 전에:
  확인: "실제 API 응답에는 어떤 필드가 있는가?"

  조치:
    1. 문서/예제에서 실제 API 응답을 확인하라
    2. 시스템이 하위에서 사용할 수 있는 모든 필드를 포함하라
    3. mock이 실제 응답 스키마와 완전히 일치하는지 검증하라

  중요:
    mock을 생성한다면, 전체 구조를 이해해야 한다
    부분 mock은 코드가 누락된 필드에 의존할 때 조용히 실패한다

  불확실하다면: 문서화된 모든 필드를 포함하라
```

## 안티패턴 5: 사후 처리로서의 통합 테스트

**위반 사례:**
```
✅ 구현 완료
❌ 테스트 없음
"테스트 준비 완료"
```

**왜 잘못됐는가:**
- 테스트는 구현의 일부이지, 선택적인 후속 작업이 아니다
- TDD였다면 이런 상황을 방지했을 것이다
- 테스트 없이 완료를 주장할 수 없다

**해결책:**
```
TDD 사이클:
1. 실패하는 테스트 작성
2. 통과하도록 구현
3. 리팩토링
4. 그런 다음 완료를 주장
```

## Mock이 너무 복잡해질 때

**경고 신호:**
- mock 설정이 테스트 로직보다 길다
- 테스트를 통과시키기 위해 모든 것을 mock한다
- Mock에 실제 컴포넌트가 가진 메서드가 없다
- mock이 변경되면 테스트가 깨진다

**담당자의 질문:** "여기서 mock을 사용해야 하는 건가요?"

**고려사항:** 실제 컴포넌트를 사용한 통합 테스트가 복잡한 mock보다 단순할 때가 많다

## TDD가 이런 안티패턴을 예방하는 방법

**TDD가 도움이 되는 이유:**
1. **테스트 먼저 작성** → 실제로 무엇을 테스트하는지 생각하게 한다
2. **실패를 확인** → 테스트가 mock이 아닌 실제 동작을 테스트함을 확인한다
3. **최소한의 구현** → 테스트 전용 메서드가 슬그머니 들어오지 않는다
4. **실제 의존성** → mock하기 전에 테스트에 실제로 무엇이 필요한지 파악한다

**mock 동작을 테스트하고 있다면, TDD를 위반한 것이다** - 실제 코드에 대해 테스트가 실패하는 것을 보지 않고 mock을 추가한 것이다.

## 빠른 참조

| 안티패턴 | 해결책 |
|---------|--------|
| 출력 대신 mock 상호작용만 단언 | 실제 컴포넌트를 테스트하거나 mock을 제거하라 |
| 프로덕션의 테스트 전용 메서드 | 테스트 유틸리티로 이동하라 |
| 이해 없이 mock 사용 | 먼저 의존성을 이해하고, 최소한으로 mock하라 |
| 불완전한 mock | 실제 API를 완전히 반영하라 |
| 사후 처리로서의 테스트 | TDD - 테스트 먼저 |
| 과도하게 복잡한 mock | 통합 테스트를 고려하라 |

## 레드 플래그

- 실제 출력이 아닌 mock 상호작용만 검증하는 단언
- 테스트 파일에서만 호출되는 메서드
- Mock 설정이 테스트의 50% 이상
- mock을 제거하면 테스트가 실패
- mock이 필요한 이유를 설명할 수 없음
- "안전을 위해" mock 사용

## 결론

**Mock은 격리를 위한 도구이지, 테스트 대상이 아니다.**

TDD를 통해 mock 동작을 테스트하고 있음이 드러난다면, 잘못된 방향으로 가고 있는 것이다.

해결책: 실제 동작을 테스트하거나, 왜 mock을 사용하는지 자문하라.
