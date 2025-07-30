# Playwright Testing Guide - Faber Extract

## Tổng quan

Playwright là một framework testing hiện đại cho web applications, hỗ trợ testing trên nhiều browsers (Chromium, Firefox, WebKit). Document này sẽ hướng dẫn cách viết và sử dụng Playwright tests trong dự án Faber Extract.

## Cấu trúc Project

```
faber-extract/
├── playwright.config.ts          # Cấu hình Playwright
├── src/test/playwright/           # Thư mục chứa tests
│   └── login/
│       └── login.spec.ts         # Test suite cho chức năng login
└── docs/front-end/playwright/
    └── playwright-testing-guide.md # Document này
```

## Cấu hình Playwright (playwright.config.ts)

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: 'src/test/playwright',    // Thư mục chứa test files
  timeout: 30000,                   // Timeout cho mỗi test (30 giây)
  retries: 1,                       // Số lần retry khi test fail
  reporter: 'html',                 // Format báo cáo (html, json, line, etc.)
  use: {
    headless: true,                 // Chạy browser ở chế độ headless
    viewport: { width: 1280, height: 720 }, // Kích thước viewport
    ignoreHTTPSErrors: true,        // Bỏ qua lỗi HTTPS
    video: 'on',                    // Ghi video cho tất cả tests
    screenshot: 'only-on-failure',  // Chụp ảnh khi test fail
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox', 
      use: { ...devices['Desktop Firefox'] },
    },
  ],
});
```

### Giải thích các config quan trọng:

- **testDir**: Định nghĩa thư mục root chứa các test files
- **timeout**: Thời gian tối đa cho mỗi test case
- **retries**: Số lần thử lại khi test bị fail (hữu ích cho các test không ổn định)
- **reporter**: Format output của test results
- **headless**: `true` = không mở browser UI, `false` = mở browser để xem
- **video/screenshot**: Hỗ trợ debug khi test fail

## Cấu trúc Test File

### 1. Import và Setup

```typescript
import { test, expect } from '@playwright/test';

test.describe('Login Functionality', () => {
  const baseUrl = 'http://localhost:8080/faber-extract';
  const loginPageUrl = `${baseUrl}/login`;
```

**Giải thích:**
- `test` và `expect` là các functions chính của Playwright
- `test.describe` nhóm các test cases liên quan với nhau
- Định nghĩa constants cho URLs để dễ maintain

### 2. Setup và Teardown

```typescript
test.beforeEach(async ({ page }, testInfo) => {
  // Webkit cần timeout dài hơn
  const isWebkit = testInfo.project.name === 'webkit';
  const formTimeout = isWebkit ? 30000 : 15000;
  
  await page.goto(loginPageUrl);
  
  // Đợi JavaScript tải và tạo form
  try {
    await page.waitForSelector('#frm_login', { timeout: formTimeout });
  } catch (e) {
    if (isWebkit) {
      // Webkit có thể cần thêm thời gian để load JS
      await page.waitForTimeout(5000);
      await page.reload();
      await page.waitForSelector('#frm_login', { timeout: formTimeout });
    } else {
      throw e;
    }
  }
  
  // Đợi thêm để đảm bảo tất cả JavaScript đã load
  await page.waitForTimeout(isWebkit ? 5000 : 3000);
});
```

**Giải thích:**
- `beforeEach` chạy trước mỗi test case
- `page.goto()` điều hướng đến URL
- `waitForSelector()` đợi element xuất hiện trong DOM
- `waitForTimeout()` đợi một khoảng thời gian cố định
- Xử lý khác biệt giữa các browsers (Webkit cần thời gian load lâu hơn)

### 3. Test Case Cơ Bản

#### Ví dụ: Test hiển thị form

```typescript
test('should display login form correctly', async ({ page }) => {
  // Kiểm tra các elements cơ bản có tồn tại
  await expect(page.locator('#frm_login')).toBeVisible();
  await expect(page.locator('#txt_email_login')).toBeVisible();
  await expect(page.locator('#txt_password')).toBeVisible();
  await expect(page.locator('button[type="submit"]')).toBeVisible();
  
  // Kiểm tra placeholder text
  await expect(page.locator('#txt_email_login')).toHaveAttribute('placeholder', 'Email');
  await expect(page.locator('#txt_password')).toHaveAttribute('placeholder', 'Password');
});
```

**Giải thích chi tiết:**

1. **`page.locator(selector)`**: Tìm element trên trang theo CSS selector
   - `#frm_login` = element có ID là "frm_login"
   - `button[type="submit"]` = button có attribute type="submit"

2. **`toBeVisible()`**: Assertion kiểm tra element có hiển thị trên trang không
   - Element phải tồn tại trong DOM
   - Element không bị ẩn (display: none, visibility: hidden, etc.)

3. **`toHaveAttribute(name, value)`**: Kiểm tra element có attribute với giá trị cụ thể
   - Ví dụ: input field có placeholder text đúng không

4. **`await expect()`**: Playwright sẽ tự động retry assertion này cho đến khi pass hoặc timeout
   - Không cần thêm waitForSelector() trong hầu hết trường hợp
   - Built-in smart waiting

**Các loại test cases khác bạn có thể viết:**
- Test validation (empty fields, invalid format)
- Test form submission (success/error scenarios)  
- Test navigation (chuyển trang, back/forward)
- Test keyboard interactions (Tab, Enter, etc.)
- Test responsive design (mobile/desktop views)

### 4. Test Navigation và Form Switching

```typescript
test('should navigate to forget password form', async ({ page }) => {
  // Click vào link "Forget Password"
  await page.click('a[href="#forget-password"]', { force: true });
  
  // Đợi transition hoàn thành
  await page.waitForTimeout(1000);
  
  // Kiểm tra form đã chuyển
  await expect(page.locator('#frm_forgot_password')).toBeVisible();
  await expect(page.locator('#frm_login')).not.toBeVisible();
});
```

### 5. Test Keyboard Navigation

```typescript
test('should handle keyboard navigation', async ({ page }, testInfo) => {
  const isFirefox = testInfo.project.name === 'firefox';
  
  // Test Tab navigation
  await page.fill('#txt_email_login', 'test@example.com');
  await page.press('#txt_email_login', 'Tab');
  
  // Kiểm tra focus đã chuyển sang password field
  await expect(page.locator('#txt_password')).toBeFocused();
  
  await page.fill('#txt_password', 'password');
  
  // Submit bằng Enter
  await page.press('#txt_password', 'Enter');
  
  // Xử lý khác biệt giữa browsers
  const waitTimeout = isFirefox ? 8000 : 3000;
  await page.waitForTimeout(waitTimeout);
});
```

**Giải thích:**
- `press()` nhấn phím keyboard
- `toBeFocused()` kiểm tra element có focus không
- Xử lý khác biệt timeout giữa các browsers

## Advanced Techniques

### 1. Xử lý BlockUI/Loading Overlays

```typescript
// Đợi blockUI biến mất
try {
  await page.waitForFunction(() => {
    const blockUI = document.querySelector('.blockUI.blockOverlay') as HTMLElement;
    return !blockUI || blockUI.style.display === 'none';
  }, { timeout: 10000 });
} catch (e) {
  console.log('BlockUI timeout - continuing test');
}
```

### 2. Force Clicks cho Overlays

```typescript
// Sử dụng force click khi có overlay che phủ
await page.click('#submit-button', { force: true });
```

### 3. Kiểm tra Multiple Conditions

```typescript
// Kiểm tra nhiều điều kiện có thể xảy ra
const hasValidationError = await page.locator('.help-block').isVisible().catch(() => false);
const hasNotification = await page.locator('.notify-osd').isVisible().catch(() => false);
const hasErrorBox = await page.locator('#error_box').isVisible().catch(() => false);
const urlChanged = page.url() !== loginPageUrl;

expect(hasValidationError || hasNotification || hasErrorBox || urlChanged).toBeTruthy();
```

### 4. Browser-Specific Handling

```typescript
test('cross-browser test', async ({ page }, testInfo) => {
  const browserName = testInfo.project.name;
  
  if (browserName === 'firefox') {
    // Firefox-specific logic
    await page.waitForTimeout(8000);
  } else if (browserName === 'webkit') {
    // Safari-specific logic
    await page.reload();
  } else {
    // Chromium default
    await page.waitForTimeout(3000);
  }
});
```

## Best Practices

### 1. Selectors

```typescript
// ✅ Tốt - Sử dụng ID hoặc data attributes
await page.locator('#submit-button');
await page.locator('[data-testid="login-form"]');

// ⚠️ Cẩn thận - CSS classes có thể thay đổi
await page.locator('.btn-primary');

// ❌ Tránh - Text content có thể thay đổi theo ngôn ngữ
await page.locator('text=Submit');
```

### 2. Assertions

```typescript
// ✅ Tốt - Specific assertions
await expect(page.locator('#error-message')).toBeVisible();
await expect(page.locator('#email')).toHaveValue('test@example.com');

// ✅ Tốt - Multiple condition check
const hasError = await page.locator('.error').count() > 0;
expect(hasError).toBeTruthy();
```

### 3. Waits

```typescript
// ✅ Tốt - Wait for specific conditions
await page.waitForSelector('#dynamic-content');
await page.waitForLoadState('networkidle');

// ⚠️ Cẩn thận - Fixed timeouts chỉ khi cần thiết
await page.waitForTimeout(1000);
```

### 4. Error Handling

```typescript
// ✅ Tốt - Graceful error handling
const hasElement = await page.locator('#optional-element').isVisible().catch(() => false);

// ✅ Tốt - Try-catch cho operations có thể fail
try {
  await page.waitForSelector('#dynamic-element', { timeout: 5000 });
} catch (e) {
  console.log('Element not found, continuing...');
}
```

## Chạy Tests

### Commands cơ bản

```bash
# Chạy tất cả tests
npx playwright test

# Chạy tests trong một file cụ thể
npx playwright test src/test/playwright/login/login.spec.ts

# Chạy với browser cụ thể
npx playwright test --project=firefox

# Chạy với UI mode (để debug)
npx playwright test --ui

# Chạy tests và mở báo cáo
npx playwright test && npx playwright show-report
```

### Debug Tests

```bash
# Chạy với headed mode (hiển thị browser)
npx playwright test --headed

# Debug một test cụ thể
npx playwright test --debug login.spec.ts

# Chạy với trace viewer
npx playwright test --trace on
```

## Troubleshooting

### 1. Timeout Issues

```typescript
// Tăng timeout cho operations chậm
await page.waitForSelector('#slow-element', { timeout: 30000 });

// Hoặc set timeout globally trong config
timeout: 60000
```

### 2. Element Not Found

```typescript
// Kiểm tra element có tồn tại không
const elementExists = await page.locator('#element').count() > 0;

// Đợi element xuất hiện
await page.waitForSelector('#element', { state: 'visible' });
```

### 3. Flaky Tests

```typescript
// Sử dụng retries
retries: 2

// Hoặc wait for stable state
await page.waitForLoadState('networkidle');
```

## Kết luận

Playwright cung cấp một framework mạnh mẽ cho việc testing web applications. Key points:

1. **Setup đúng cách**: Config timeout, retries, browsers phù hợp
2. **Selectors ổn định**: Sử dụng ID, data attributes thay vì CSS classes
3. **Xử lý async**: Luôn await các operations và handle timeouts
4. **Cross-browser testing**: Test trên nhiều browsers và handle differences
5. **Error handling**: Graceful handling cho các edge cases

Document này sẽ được cập nhật khi có thêm test cases và best practices mới.
