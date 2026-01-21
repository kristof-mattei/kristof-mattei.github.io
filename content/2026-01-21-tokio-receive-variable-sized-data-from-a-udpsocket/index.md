+++
title = "Tokio: receive variable-sized data from a UdpSocket"
date = 2026-01-21 15:49:01-07:00

[taxonomies]
categories = ["network", "rust", "tokio", "libc"]

[extra]
author = "Kristof"
+++

Imagine you're listening on a [`tokio::net::UdpSocket`](https://docs.rs/tokio/latest/tokio/net/struct.UdpSocket.html) and you know you're going to receive a packet, but don't know the size.

The first option was to find out what the other party might send, but that information might not always be available.

Another way might be to dynamically allocate a buffer based on the incoming packet size:

```rust
use tokio::net::UdpSocket;

async fn get_buffer_size(socket: &UdpSocket) -> Result<usize, std::io::Error> {
    let result = socket
        .async_io(Interest::READABLE, || {
            let mut mini_buf = [0_u8; 32];

            // SAFETY: libc call
            let result = unsafe {
                libc::recv(
                    socket.as_raw_fd(),
                    mini_buf.as_mut_ptr().cast(),
                    mini_buf.len(),
                    libc::MSG_PEEK | libc::MSG_TRUNC,
                )
            };

            if result < 0 {
                return Err(std::io::Error::last_os_error());
            }

            Ok(result.unsigned_abs())
        })
        .await?;

    Ok(result)
}
```

Couple of things to note:

* We're using [`MSG_PEEK`](https://www.man7.org/linux/man-pages/man2/recv.2.html#:~:text=with%20such%20protocols.-,MSG_PEEK,-This%20flag%20causes) to ensure we don't pull the data from the buffer, we just look at it.
* [`MSG_TRUNC`](https://www.man7.org/linux/man-pages/man2/recv.2.html#:~:text=the%20same%20data.-,MSG_TRUNC,-%28since%20Linux%202.2) to ensure we get back the total number of bytes pending in the kernel's buffer, not just the amount written to OUR buffer
* We stack-allocated the buffer to avoid the allocation hit every time.
