# lib.rs

```rust

use crossbeam_channel::{unbounded, Receiver, Sender};
use pyo3::{prelude::*, types::PyDict};
use std::sync::Arc;
use std::time::{Duration, Instant};

/// A handle that allows sending messages **from Python to Rust**.
///
/// Exposed to Python as a class. Wraps a `Sender<Py<PyAny>>`.
#[pyclass]
struct SenderHandle {
    tx: Arc<Sender<Py<PyAny>>>,
}

#[pymethods]
impl SenderHandle {
    /// Send a Python object to the Rust event loop.
    ///
    /// # Errors
    /// Returns a `PyRuntimeError` if the send operation fails.
    fn send(&self, msg: Py<PyAny>) -> PyResult<()> {
        self.tx
            .send(msg)
            .map_err(|e| pyo3::exceptions::PyRuntimeError::new_err(format!("send failed: {e}")))
    }
}

/// A handle that allows receiving messages **from Rust to Python**.
///
/// Exposed to Python as a class. Wraps a `Receiver<Py<PyAny>>`.
#[pyclass]
struct ReceiverHandle {
    rx: Arc<Receiver<Py<PyAny>>>,
}

#[pymethods]
impl ReceiverHandle {
    /// Attempt to receive a message without blocking.
    ///
    /// Returns `Some(PyAny)` if a message is available, otherwise `None`.
    fn recv(&self) -> Option<Py<PyAny>> {
        self.rx.try_recv().ok()
    }

    /// Wait until a message is available and return it.
    ///
    /// This method blocks the current thread until a message is received.
    fn recv_blocking(&self) -> Option<Py<PyAny>> {
        self.rx.recv().ok()
    }
}

/// Run the GUI loop that integrates Rust and Python.
///
/// - Spawns a Python thread that sets up an asyncio event loop.
/// - Creates two channels:
///   - Python → Rust
///   - Rust → Python
/// - Periodically injects a `"ping"` IPC event into the Python queue.
///
/// # Arguments
/// - `py_event_loop`: A Python asyncio event loop instance.
/// - `pyevent_to_rust_queue`: Python coroutine to handle sending events to Rust.
/// - `rust_to_py_ipc`: Python coroutine to handle receiving IPC events from Rust.
/// - `rust_loop_event_handler`: Python callback invoked when Rust processes a Python event.
#[pyfunction]
fn run_gui(
    py_event_loop: Py<PyAny>,
    pyevent_to_rust_queue: Py<PyAny>,
    rust_to_py_ipc: Py<PyAny>,
    rust_loop_event_handler: Py<PyAny>,
) -> PyResult<()> {
    let (tx_py_to_rust, rx_py_to_rust) = unbounded::<Py<PyAny>>();
    let (tx_rust_to_py, rx_rust_to_py) = unbounded::<Py<PyAny>>();

    let tx_py_to_rust = Arc::new(tx_py_to_rust);
    let rx_py_to_rust = Arc::new(rx_py_to_rust);
    let tx_rust_to_py = Arc::new(tx_rust_to_py);
    let rx_rust_to_py = Arc::new(rx_rust_to_py);

    let _py_thread = spawn_py_event_loop(
        py_event_loop,
        pyevent_to_rust_queue,
        rust_to_py_ipc,
        tx_py_to_rust.clone(),
        rx_rust_to_py.clone(),
    )?;

    let mut last_ipc = Instant::now();

    // Main GUI loop
    loop {
        // Process incoming Python messages
        while let Ok(msg) = rx_py_to_rust.try_recv() {
            Python::with_gil(|py| {
                let handler = rust_loop_event_handler.bind(py);
                if let Err(e) = handler.call1((msg,)) {
                    eprintln!("Error calling handler: {:?}", e);
                }
            });
        }

        // Inject IPC event every 3 seconds
        if last_ipc.elapsed() >= Duration::from_secs(3) {
            simulate_ipc_event(tx_rust_to_py.clone());
            last_ipc = Instant::now();
        }
    }
}

/// Simulate an IPC event by sending a `"ping"` message into the Rust → Python queue.
fn simulate_ipc_event(tx_rust_to_py: Arc<Sender<Py<PyAny>>>) {
    Python::with_gil(|py| {
        let msg = PyDict::new(py);
        msg.set_item("cmd", "ping").unwrap();
        msg.set_item("payload", "hello from Rust").unwrap();
        let py_msg = msg.into();
        let _ = tx_rust_to_py.send(py_msg);
    });
}

/// Spawn the Python event loop thread.
///
/// This function:
/// - Configures the provided asyncio loop as the default loop.
/// - Exposes `SenderHandle` (Python → Rust) and `ReceiverHandle` (Rust → Python).
/// - Registers Python tasks for event passing.
/// - Runs the asyncio loop forever.
///
/// # Returns
/// A `JoinHandle` for the spawned Python thread.
///
/// # Errors
/// Returns an error if Python initialization fails.
fn spawn_py_event_loop(
    py_event_loop: Py<PyAny>,
    pyevent_to_rust_queue: Py<PyAny>,
    rust_to_py_ipc: Py<PyAny>,
    tx_from_py_to_rust: Arc<Sender<Py<PyAny>>>,
    rx_from_rust_to_py: Arc<Receiver<Py<PyAny>>>,
) -> anyhow::Result<std::thread::JoinHandle<()>> {
    let handle = std::thread::spawn(move || {
        Python::with_gil(|py| -> PyResult<()> {
            println!("[PY] Python-Thread started");

            let asyncio = py.import("asyncio")?;
            let loop_obj = py_event_loop.bind(py);

            // Set the event loop
            asyncio.call_method1("set_event_loop", (loop_obj.clone(),))?;

            // Create sender and receiver handles
            let sender = Py::new(
                py,
                SenderHandle {
                    tx: tx_from_py_to_rust.clone(),
                },
            )?;
            let receiver = Py::new(
                py,
                ReceiverHandle {
                    rx: rx_from_rust_to_py.clone(),
                },
            )?;

            // Register Python tasks
            let py_events = pyevent_to_rust_queue.call1(py, (sender,))?;
            let py_ipc = rust_to_py_ipc.call1(py, (receiver,))?;

            loop_obj.call_method1("create_task", (py_events,))?;
            loop_obj.call_method1("create_task", (py_ipc,))?;

            // Run the asyncio loop forever
            loop_obj.call_method0("run_forever")?;

            Ok(())
        })
        .unwrap_or_else(|e| eprintln!("Python thread error: {:?}", e));
    });

    Ok(handle)
}

/// Python module initializer.
///
/// Exposes:
/// - `SenderHandle`
/// - `ReceiverHandle`
/// - `run_gui()`
#[pymodule]
fn pygui(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_class::<SenderHandle>()?;
    m.add_class::<ReceiverHandle>()?;
    m.add_function(wrap_pyfunction!(run_gui, m)?)?;
    Ok(())
}


```
# pygui.pyi
```python

# Stubs for the Rust module `pygui`
# Auto-generated based on PyO3 definitions

from typing import Any, Optional

class SenderHandle:
    """
    A handle for sending messages from Python to the Rust event loop.
    Wraps a Rust channel Sender.
    """

    def send(self, msg: Any) -> None:
        """
        Send a Python object to Rust.

        :param msg: Any Python object to be passed into Rust.
        :raises RuntimeError: if sending fails.
        """
        ...

class ReceiverHandle:
    """
    A handle for receiving messages from Rust to Python.
    Wraps a Rust channel Receiver.
    """

    def recv(self) -> Optional[Any]:
        """
        Non-blocking receive.

        :return: The next message if available, otherwise None.
        """
        ...

    def recv_blocking(self) -> Optional[Any]:
        """
        Blocking receive.

        :return: Waits until a message is available and returns it.
        """
        ...

def run_gui(
    py_event_loop: Any,
    pyevent_to_rust_queue: Any,
    rust_to_py_ipc: Any,
    rust_loop_event_handler: Any,
) -> None:
    """
    Start the integrated Rust <-> Python GUI loop.

    :param py_event_loop: A Python asyncio event loop instance.
    :param pyevent_to_rust_queue: Coroutine factory handling Python -> Rust events.
    :param rust_to_py_ipc: Coroutine factory handling Rust -> Python IPC messages.
    :param rust_loop_event_handler: Python callback invoked for messages received in Rust.
    """
    ...


```
# main.py
```python

"""
Example usage of the `pygui` Rust extension.

This script demonstrates:
- Sending messages from Python -> Rust via SenderHandle
- Receiving messages from Rust (event loop handler)
- Receiving periodic IPC events from Rust via ReceiverHandle
"""

import asyncio
import random
from pygui import ReceiverHandle, SenderHandle, run_gui


async def queue_watcher(sender: SenderHandle):
    """
    Coroutine that periodically sends random integers to Rust.
    """
    while True:
        d = random.randint(555, 999)
        print(f"[PY] send to Rust: {d}", flush=True)
        sender.send({"data": d})  # Send Python object to Rust
        await asyncio.sleep(1)


async def handle_rust_eventloop(data):
    """
    Callback invoked by Rust when a message is dispatched
    from the Rust loop event handler.
    """
    print(f"[PY] recv from Rust eventloop: {data}", flush=True)


async def ipc_handler(receiver: ReceiverHandle):
    """
    Coroutine that continuously polls the Rust -> Python IPC channel.

    Uses blocking receive to wait for messages.
    """
    while True:
        msg = receiver.recv_blocking()
        if msg:
            print(f"[PY] recv from Rust IPC: {msg}", flush=True)
        await asyncio.sleep(0.1)


def main():
    """
    Entry point: initializes asyncio loop and runs the Rust GUI bridge.
    """
    try:
        loop_obj = asyncio.new_event_loop()
        asyncio.set_event_loop(loop_obj)

        # Run integrated Rust <-> Python loop
        run_gui(loop_obj, queue_watcher, ipc_handler, handle_rust_eventloop)
    except Exception as e:
        print(f"Error: {e}", flush=True)
    finally:
        print("Shutting down", flush=True)


if __name__ == "__main__":
    main()


```
