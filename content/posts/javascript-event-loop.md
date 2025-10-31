---
title: "Event Loop là gì trong JavaScript? "
date: 2025-10-23
draft: false
tags: ["JavaScript", "Event Loop", "Async", "Programming", "Web Development"]
description: "Tìm hiểu chi tiết về Event Loop trong JavaScript - cơ chế quan trọng giúp JavaScript xử lý các tác vụ bất đồng bộ một cách hiệu quả"
---

## Giới thiệu

JavaScript là một ngôn ngữ lập trình **single-threaded** (đơn luồng), nhưng tại sao chúng ta vẫn có thể thực hiện các tác vụ bất đồng bộ như gọi API, xử lý event, hay setTimeout? Câu trả lời nằm ở **Event Loop** - một cơ chế quan trọng và thú vị của JavaScript runtime.

Trong bài viết này, chúng ta sẽ khám phá chi tiết về Event Loop, cách nó hoạt động, và tại sao việc hiểu rõ về nó lại quan trọng đối với mọi JavaScript developer.

## Event Loop là gì?

**Event Loop** là cơ chế giúp JavaScript xử lý các tác vụ bất đồng bộ (asynchronous) mặc dù JavaScript chỉ có một luồng thực thi duy nhất. Nó giống như một "người điều phối giao thông" kiểm soát việc thực thi code một cách có trật tự.

### Tại sao cần Event Loop?

JavaScript được thiết kế để chạy trên trình duyệt và tương tác với người dùng. Nếu không có Event Loop:

- Giao diện sẽ bị "đóng băng" khi thực hiện các tác vụ tốn thời gian
- Không thể xử lý nhiều sự kiện cùng lúc
- Trải nghiệm người dùng sẽ rất tệ

## Kiến trúc JavaScript Runtime

Để hiểu Event Loop, trước tiên chúng ta cần biết về kiến trúc của JavaScript Runtime:

### 1. Call Stack (Ngăn xếp gọi hàm)

```javascript
function first() {
    console.log('First function');
    second();
}

function second() {
    console.log('Second function');
    third();
}

function third() {
    console.log('Third function');
}

first();
```

Call Stack hoạt động theo nguyên tắc LIFO (Last In, First Out):
1. `first()` được đưa vào stack
2. `second()` được đưa vào stack
3. `third()` được đưa vào stack
4. `third()` hoàn thành, bị loại khỏi stack
5. `second()` hoàn thành, bị loại khỏi stack
6. `first()` hoàn thành, bị loại khỏi stack

### 2. Web APIs

Trình duyệt cung cấp các Web APIs để xử lý các tác vụ bất đồng bộ:
- **DOM API**: Xử lý sự kiện DOM
- **Timer API**: setTimeout, setInterval
- **HTTP API**: XMLHttpRequest, fetch
- **Promise API**: Promise resolution

### 3. Callback Queue (Hàng đợi callback)

Chứa các callback function đang chờ được thực thi:

```javascript
setTimeout(() => {
    console.log('Timer callback');
}, 0);

console.log('Synchronous code');
```

### 4. Microtask Queue

Hàng đợi ưu tiên cao hơn Callback Queue, chứa:
- Promise callbacks (.then, .catch, .finally)
- queueMicrotask()
- MutationObserver callbacks

## Cách Event Loop hoạt động

Event Loop liên tục thực hiện các bước sau:

### Bước 1: Kiểm tra Call Stack
```javascript
console.log('Start');

setTimeout(() => {
    console.log('Timeout');
}, 0);

Promise.resolve().then(() => {
    console.log('Promise');
});

console.log('End');
```

### Bước 2: Xử lý Microtasks
Event Loop ưu tiên xử lý hết tất cả microtasks trước:

```javascript
setTimeout(() => console.log('1'), 0);

Promise.resolve().then(() => console.log('2'));
Promise.resolve().then(() => console.log('3'));

setTimeout(() => console.log('4'), 0);

// Output: 2, 3, 1, 4
```

### Bước 3: Xử lý Macrotasks
Sau khi microtask queue trống, xử lý một macrotask:

```javascript
console.log('=== Start ===');

setTimeout(() => {
    console.log('Timer 1');
    Promise.resolve().then(() => console.log('Promise in Timer 1'));
}, 0);

setTimeout(() => {
    console.log('Timer 2');
}, 0);

Promise.resolve().then(() => {
    console.log('Promise 1');
    Promise.resolve().then(() => console.log('Promise 2'));
});

console.log('=== End ===');

// Output:
// === Start ===
// === End ===
// Promise 1
// Promise 2
// Timer 1
// Promise in Timer 1
// Timer 2
```

## Ví dụ thực tế và phức tạp

### Ví dụ 1: Kết hợp setTimeout và Promise

```javascript
async function complexExample() {
    console.log('1');
    
    setTimeout(() => console.log('2'), 0);
    
    await new Promise(resolve => {
        console.log('3');
        resolve();
    });
    
    console.log('4');
    
    setTimeout(() => console.log('5'), 0);
    
    Promise.resolve().then(() => console.log('6'));
    
    console.log('7');
}

complexExample();

// Output: 1, 3, 4, 7, 6, 2, 5
```

### Ví dụ 2: Event handling và Event Loop

```javascript
// HTML: <button id="btn">Click me</button>

document.getElementById('btn').addEventListener('click', () => {
    console.log('Click event');
    
    Promise.resolve().then(() => console.log('Promise in click'));
    
    setTimeout(() => console.log('Timeout in click'), 0);
});

// Khi click button:
// Click event
// Promise in click
// Timeout in click
```

### Ví dụ 3: Nested Promises và setTimeout

```javascript
setTimeout(() => {
    console.log('Timer 1');
    
    Promise.resolve().then(() => {
        console.log('Promise 1');
        
        Promise.resolve().then(() => {
            console.log('Nested Promise 1');
        });
        
        setTimeout(() => {
            console.log('Nested Timer 1');
        }, 0);
    });
    
    setTimeout(() => {
        console.log('Timer 1 - Inner');
    }, 0);
}, 0);

setTimeout(() => {
    console.log('Timer 2');
}, 0);

// Output:
// Timer 1
// Promise 1
// Nested Promise 1
// Timer 1 - Inner
// Timer 2
// Nested Timer 1
```

## Những lỗi thường gặp liên quan đến Event Loop

### 1. Blocking Event Loop

```javascript
// ❌ Không nên làm - blocking code
function heavyTask() {
    let result = 0;
    for (let i = 0; i < 10000000000; i++) {
        result += i;
    }
    return result;
}

console.log('Before heavy task');
heavyTask(); // Chặn Event Loop
console.log('After heavy task');
```

```javascript
// ✅ Nên làm - non-blocking
function heavyTaskAsync(callback) {
    let result = 0;
    let i = 0;
    
    function chunk() {
        const start = Date.now();
        while (i < 10000000000 && Date.now() - start < 5) {
            result += i++;
        }
        
        if (i < 10000000000) {
            setTimeout(chunk, 0);
        } else {
            callback(result);
        }
    }
    
    chunk();
}

console.log('Before heavy task');
heavyTaskAsync((result) => {
    console.log('Heavy task completed:', result);
});
console.log('After heavy task');
```

### 2. Promise Hell và Event Loop

```javascript
// ❌ Hiểu sai về thứ tự thực thi
Promise.resolve().then(() => {
    console.log('Promise 1');
    return new Promise(resolve => {
        setTimeout(() => {
            console.log('Promise 1 - Timeout');
            resolve();
        }, 0);
    });
}).then(() => {
    console.log('Promise 2');
});

setTimeout(() => {
    console.log('Independent Timeout');
}, 0);

// Output:
// Promise 1
// Independent Timeout
// Promise 1 - Timeout
// Promise 2
```

## Debugging Event Loop

### 1. Sử dụng console.trace()

```javascript
function debugEventLoop() {
    console.log('Sync code');
    console.trace('Sync trace');
    
    setTimeout(() => {
        console.log('Timeout');
        console.trace('Timeout trace');
    }, 0);
    
    Promise.resolve().then(() => {
        console.log('Promise');
        console.trace('Promise trace');
    });
}

debugEventLoop();
```

### 2. Performance monitoring

```javascript
function measureEventLoop() {
    const start = performance.now();
    
    // Thêm nhiều tasks
    for (let i = 0; i < 100; i++) {
        setTimeout(() => {
            if (i === 99) {
                console.log('All timeouts completed in:', 
                           performance.now() - start, 'ms');
            }
        }, 0);
    }
    
    // Thêm microtasks
    for (let i = 0; i < 100; i++) {
        Promise.resolve().then(() => {
            if (i === 99) {
                console.log('All promises completed in:', 
                           performance.now() - start, 'ms');
            }
        });
    }
}

measureEventLoop();
```

## Event Loop trong các môi trường khác nhau

### Node.js Event Loop

```javascript
// Node.js có 6 phases trong Event Loop
const fs = require('fs');

console.log('Start');

// Timer phase
setTimeout(() => console.log('Timer'), 0);
setImmediate(() => console.log('Immediate'));

// I/O phase
fs.readFile(__filename, () => {
    console.log('File read');
    
    setTimeout(() => console.log('Timer in I/O'), 0);
    setImmediate(() => console.log('Immediate in I/O'));
});

// Close callbacks phase
const server = require('http').createServer();
server.on('close', () => console.log('Server closed'));
server.close();

console.log('End');
```

### Web Workers và Event Loop

```javascript
// main.js
const worker = new Worker('worker.js');

worker.postMessage({task: 'heavy-computation'});

worker.onmessage = function(e) {
    console.log('Result from worker:', e.data);
};

// Event Loop không bị block bởi worker
setTimeout(() => {
    console.log('Main thread timer');
}, 100);
```

```javascript
// worker.js
self.onmessage = function(e) {
    if (e.data.task === 'heavy-computation') {
        let result = 0;
        for (let i = 0; i < 10000000000; i++) {
            result += i;
        }
        self.postMessage(result);
    }
};
```

## Best Practices

### 1. Tránh blocking Event Loop

```javascript
// ✅ Chia nhỏ tasks
function processLargeArray(array, batchSize = 1000) {
    let index = 0;
    
    function processBatch() {
        const start = index;
        const end = Math.min(index + batchSize, array.length);
        
        for (let i = start; i < end; i++) {
            // Process array[i]
            array[i] = array[i] * 2;
        }
        
        index = end;
        
        if (index < array.length) {
            setTimeout(processBatch, 0);
        }
    }
    
    processBatch();
}
```

### 2. Sử dụng đúng async/await

```javascript
// ✅ Hiểu rõ về async/await và Event Loop
async function optimizedAsyncCode() {
    console.log('1');
    
    // Parallel execution
    const promise1 = fetch('/api/data1');
    const promise2 = fetch('/api/data2');
    
    console.log('2');
    
    const [result1, result2] = await Promise.all([promise1, promise2]);
    
    console.log('3');
    
    return {result1, result2};
}
```

### 3. Monitoring performance

```javascript
// Performance monitoring
function monitorEventLoop() {
    let lastTime = performance.now();
    
    function check() {
        const currentTime = performance.now();
        const delay = currentTime - lastTime - 16; // Expected 16ms for 60fps
        
        if (delay > 5) {
            console.warn(`Event Loop delay: ${delay.toFixed(2)}ms`);
        }
        
        lastTime = currentTime;
        requestAnimationFrame(check);
    }
    
    requestAnimationFrame(check);
}

monitorEventLoop();
```

## Kết luận

Event Loop là trái tim của JavaScript, giúp ngôn ngữ này có thể xử lý các tác vụ bất đồng bộ một cách mượt mà. Việc hiểu rõ Event Loop giúp bạn:

1. **Viết code hiệu quả hơn**: Tránh blocking Event Loop
2. **Debug dễ dàng hơn**: Hiểu thứ tự thực thi code
3. **Tối ưu performance**: Sử dụng đúng async patterns
4. **Tránh race conditions**: Hiểu rõ timing của các operations

Hãy thực hành với các ví dụ trong bài viết này và tiếp tục khám phá các khái niệm nâng cao khác của JavaScript!

## Tài liệu tham khảo

- [MDN: Concurrency model and Event Loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop)
- [Node.js Event Loop](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
- [JavaScript Event Loop Visualization](http://latentflip.com/loupe/)

---

*Bài viết này là một phần trong series "Hiểu sâu về JavaScript". Hãy theo dõi blog để cập nhật những bài viết mới nhất về lập trình web!*