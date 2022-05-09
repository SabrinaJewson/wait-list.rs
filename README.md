# wait-list

This crate provides `WaitList`, the most fundamental type for async synchronization. `WaitList`
is implemented as an intrusive linked list of futures.

## Feature flags

- `std`: Implements the `Lock` traits on locks from the standard library.
- `lock_api_04`: Implements the `Lock` traits on locks from [`lock_api`] v0.4. This enables
integration of crates like [`parking_lot`], [`spin`] and [`usync`].

## Example

A thread-safe unfair async mutex.

```rust
use std::cell::UnsafeCell;
use std::ops::Deref;
use std::ops::DerefMut;
use wait_list::WaitList;

pub struct Mutex<T> {
    data: UnsafeCell<T>,
    waiters: WaitList<std::sync::Mutex<bool>, (), ()>,
}

unsafe impl<T> Sync for Mutex<T> {}

impl<T> Mutex<T> {
    pub fn new(data: T) -> Self {
        Self {
            data: UnsafeCell::new(data),
            waiters: WaitList::new(std::sync::Mutex::new(false)),
        }
    }
    pub async fn lock(&self) -> Guard<'_, T> {
        loop {
            wait_list::await_send_after_poll!({
                let mut waiters = self.waiters.lock_exclusive();

                if !*waiters.guard {
                    *waiters.guard = true;
                    return Guard { mutex: self };
                }

                waiters.wait_hrtb((), |mut list, ()| { let _ = list.wake_one(()); })
            });
        }
    }
}

pub struct Guard<'mutex, T> {
    mutex: &'mutex Mutex<T>,
}

impl<T> Deref for Guard<'_, T> {
    type Target = T;
    fn deref(&self) -> &Self::Target {
        unsafe { &*self.mutex.data.get() }
    }
}
impl<T> DerefMut for Guard<'_, T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        unsafe { &mut *self.mutex.data.get() }
    }
}

impl<T> Drop for Guard<'_, T> {
    fn drop(&mut self) {
        let mut waiters = self.mutex.waiters.lock_exclusive();
        *waiters.guard = false;
        let _ = waiters.wake_one(());
    }
}
#
```

[`lock_api`]: https://docs.rs/lock_api
[`parking_lot`]: https://docs.rs/parking_lot
[`spin`]: https://docs.rs/spin
[`usync`]: https://docs.rs/usync

License: MIT