# Generic Runtime 


```rust

use std::{fmt::Debug, net::SocketAddr, sync::Arc};
use tokio::{
    io::{AsyncBufReadExt, AsyncWriteExt, BufReader},
    net::{TcpListener, TcpStream},
    runtime::{Handle, Runtime as TokioRt},
    sync::mpsc,
};

use anyhow::Result;

/// Marker trait for events handled by the runtime and TCP server.
///
/// Requirements:
/// - `Debug` for logging.
/// - `Clone` to duplicate events.
/// - `Send` + `'static` to allow safe async usage.
pub trait UserEvent: Debug + Clone + Send + 'static {}
impl<T: Debug + Clone + Send + 'static> UserEvent for T {}

/// Abstraction for a runtime handle that can spawn tasks and send events.
///
/// This trait is implemented by types that provide access to the runtime,
/// without owning it directly.
pub trait RuntimeHandle<T: UserEvent>: Debug + Clone + Send + Sync + 'static {
    /// The associated runtime type.
    type Runtime: Runtime<T, Handle = Self>;

    /// Spawn a new asynchronous task on the runtime.
    fn spawn<F>(&self, fut: F) -> Result<()>
    where
        F: std::future::Future<Output = ()> + Send + 'static;

    /// Send an event into the runtime’s event system.
    #[allow(dead_code)]
    fn send(&self, event: T) -> Result<()>;
}

/// Trait defining a generic TCP server for events.
///
/// Provides methods to start/stop the server and to send/receive events.
pub trait TCPServer<T: UserEvent>: Debug + Send + Sync + 'static {
    /// The runtime type that drives this server.
    type Runtime: Runtime<T>;

    /// Start the server on its configured address using a Tokio handle.
    fn start(&self, handle: &Handle) -> Result<()>;

    /// Stop the server gracefully.
    fn stop(&self) -> Result<()>;

    /// Send an event into the server’s event channel.
    fn send(&self, event: T) -> Result<()>;

    /// Receive an event from the server (non-blocking).
    fn receive(&self) -> Result<T>;
}

/// Lightweight handle to a TCP server.
///
/// Provides event send/receive without full server ownership.
pub trait TCPServerHandle<T: UserEvent>: Debug + Clone + Send + Sync + 'static {
    /// The associated server implementation.
    type Server: TCPServer<T>;

    /// Send an event to the underlying server.
    fn send(&self, event: T) -> Result<()>;

    /// Receive an event from the server (non-blocking).
    #[allow(dead_code)]
    fn receive(&self) -> Result<T>;
}

/// Defines a runtime abstraction that integrates a TCP server.
///
/// Provides initialization and access to its server.
pub trait Runtime<T: UserEvent>: Debug + Send + Sync + 'static {
    /// Handle type for spawning tasks and sending events.
    type Handle: RuntimeHandle<T, Runtime = Self>;

    /// TCP server type bound to this runtime.
    type TCPServer: TCPServer<T, Runtime = Self>;

    /// Initialize a runtime from its handle.
    fn init(handle: Self::Handle) -> Self;

    /// Access the runtime’s TCP server.
    fn tcp_server(&self) -> &Self::TCPServer;
}

/// Example application event type: wraps a simple string message.
#[allow(dead_code)]
#[derive(Debug, Clone)]
pub struct AppEvent(pub String);

/// TCP server implementation for `AppEvent`.
///
/// Handles TCP connections, spawns per-client handlers,
/// and forwards messages as `AppEvent`s.
#[derive(Debug)]
pub struct AppTCPServer {
    addr: SocketAddr,
    sender: mpsc::UnboundedSender<AppEvent>,
    receiver: Arc<tokio::sync::Mutex<mpsc::UnboundedReceiver<AppEvent>>>,
}

impl AppTCPServer {
    /// Create a new `AppTCPServer` bound to the given address.
    fn new(addr: SocketAddr) -> Self {
        let (tx, rx) = mpsc::unbounded_channel();
        Self {
            addr,
            sender: tx,
            receiver: Arc::new(tokio::sync::Mutex::new(rx)),
        }
    }
}

impl TCPServer<AppEvent> for AppTCPServer {
    type Runtime = AppRuntime;

    fn start(&self, handle: &Handle) -> Result<()> {
        let addr = self.addr;
        let sender = self.sender.clone();

        handle.spawn(async move {
            let listener = TcpListener::bind(addr).await.unwrap();
            println!("[TCPServer] Listening on {}", addr);

            loop {
                match listener.accept().await {
                    Ok((socket, peer)) => {
                        println!("[TCPServer] Client connected: {}", peer);
                        let tx = sender.clone();
                        tokio::spawn(handle_client(socket, tx));
                    }
                    Err(e) => {
                        eprintln!("[TCPServer] accept error: {}", e);
                        break;
                    }
                }
            }
        });
        Ok(())
    }

    fn stop(&self) -> Result<()> {
        println!("[TCPServer] stopped.");
        Ok(())
    }

    fn send(&self, event: AppEvent) -> Result<()> {
        self.sender
            .send(event)
            .map_err(|e| anyhow::anyhow!(e.to_string()))
    }

    fn receive(&self) -> Result<AppEvent> {
        let mut rx = self
            .receiver
            .try_lock()
            .map_err(|_| anyhow::anyhow!("Receiver busy".to_string()))?;
        rx.try_recv()
            .map_err(|_| anyhow::anyhow!("No event available".to_string()))
    }
}

/// Handle an individual TCP client connection.
///
/// Reads lines from the socket, forwards them as `AppEvent`s,
/// and echoes responses back to the client.
async fn handle_client(socket: TcpStream, tx: mpsc::UnboundedSender<AppEvent>) {
    let (reader, mut writer) = socket.into_split();
    let mut buf_reader = BufReader::new(reader);
    let mut line = String::new();

    loop {
        line.clear();
        let n = buf_reader.read_line(&mut line).await.unwrap();
        if n == 0 {
            break;
        }
        let msg = line.trim().to_string();
        println!("[TCPServer] received from client: {}", msg);
        let _ = tx.send(AppEvent(msg.clone()));
        let _ = writer
            .write_all(format!("echo: {}\n", msg).as_bytes())
            .await;
    }
}

/// Handle type for `AppTCPServer`.
///
/// Provides safe external access to send and receive events.
#[derive(Debug, Clone)]
pub struct AppTCPServerHandle {
    inner: Arc<AppTCPServer>,
}

impl TCPServerHandle<AppEvent> for AppTCPServerHandle {
    type Server = AppTCPServer;

    fn send(&self, event: AppEvent) -> Result<()> {
        self.inner.send(event)
    }
    fn receive(&self) -> Result<AppEvent> {
        self.inner.receive()
    }
}

/// Application runtime that integrates a TCP server and Tokio runtime.
#[allow(dead_code)]
#[derive(Debug)]
pub struct AppRuntime {
    handle: AppRuntimeHandle,
    server: Arc<AppTCPServer>,
    tokio_rt: Arc<TokioRt>,
}

/// Handle for `AppRuntime`.
///
/// Used to spawn async tasks and send events to the server.
#[derive(Debug, Clone)]
pub struct AppRuntimeHandle {
    server_handle: AppTCPServerHandle,
    tokio_rt: Arc<TokioRt>,
}

impl RuntimeHandle<AppEvent> for AppRuntimeHandle {
    type Runtime = AppRuntime;

    fn spawn<F>(&self, fut: F) -> Result<()>
    where
        F: std::future::Future<Output = ()> + Send + 'static,
    {
        self.tokio_rt.spawn(fut);
        Ok(())
    }

    fn send(&self, event: AppEvent) -> Result<()> {
        self.server_handle.send(event)
    }
}

impl Runtime<AppEvent> for AppRuntime {
    type Handle = AppRuntimeHandle;
    type TCPServer = AppTCPServer;

    fn init(handle: Self::Handle) -> Self {
        let server = handle.server_handle.inner.clone();
        let tokio_rt = handle.tokio_rt.clone();
        Self {
            handle,
            server,
            tokio_rt,
        }
    }

    fn tcp_server(&self) -> &Self::TCPServer {
        &self.server
    }
}

impl AppRuntime {
    /// Create a new runtime handle with a fresh TCP server at the given address.
    pub fn new(rt: TokioRt, addr: SocketAddr) -> AppRuntimeHandle {
        let server = Arc::new(AppTCPServer::new(addr));
        let tokio_rt = Arc::new(rt);

        let server_handle = AppTCPServerHandle {
            inner: server.clone(),
        };

        AppRuntimeHandle {
            server_handle,
            tokio_rt,
        }
    }
}

/// Setup function to demonstrate usage.
///
/// - Builds a multi-threaded Tokio runtime.
/// - Creates and initializes an `AppRuntime`.
/// - Starts the TCP server on `127.0.0.1:8080`.
/// - Spawns an example async task.
/// - Keeps the runtime alive for 30 seconds, then stops the server.
pub fn setup() -> Result<()> {
    let rt = tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()?;

    let addr: SocketAddr = "127.0.0.1:8080".parse().unwrap();
    let handle = AppRuntime::new(rt, addr);
    let runtime = AppRuntime::init(handle.clone());

    // Run Tokio runtime in a background thread
    let rt_for_thread = runtime.tokio_rt.clone();
    std::thread::spawn(move || {
        rt_for_thread.block_on(async {
            println!("[Tokio] Runtime running in background thread...");
            loop {
                tokio::time::sleep(std::time::Duration::from_secs(3600)).await;
            }
        });
    });

    // Start TCP server
    runtime
        .tcp_server()
        .start(&runtime.tokio_rt.handle().clone())?;

    // Spawn demo task
    handle.spawn(async {
        println!("[Tokio] async task running...");
    })?;

    println!("Server running - try `nc 127.0.0.1 8080` and type messages.");

    // Keep alive for 30 seconds
    std::thread::sleep(std::time::Duration::from_secs(30));

    // Stop server
    runtime.tcp_server().stop()?;
    Ok(())
}




```
