use anyhow::Result;
use clap::{Parser, Subcommand};
use std::{io::{self, BufRead}, net::SocketAddr, sync::Arc};
use tokio::{
    io::{AsyncBufReadExt, AsyncWriteExt, BufReader},
    net::{TcpListener, TcpStream},
    sync::broadcast,
};

#[derive(Parser)]
#[command(name="mini-chat", version, about="Simple TCP chat (server/client)")]
struct Cli {
    #[command(subcommand)]
    cmd: Mode,
}

#[derive(Subcommand)]
enum Mode {
    /// Run chat server on given address (e.g. 0.0.0.0:9000)
    Server { addr: String },
    /// Connect as client to server (e.g. 127.0.0.1:9000)
    Client { addr: String, #[arg(short, long, default_value="")] nick: String },
}

#[tokio::main]
async fn main() -> Result<()> {
    let cli = Cli::parse();
    match cli.cmd {
        Mode::Server { addr } => run_server(addr).await?,
        Mode::Client { addr, nick } => run_client(addr, nick).await?,
    }
    Ok(())
}

async fn run_server(addr: String) -> Result<()> {
    let listener = TcpListener::bind(&addr).await?;
    println!("Server listening on {}", addr);
    let (tx, _rx) = broadcast::channel::<String>(512);
    let tx = Arc::new(tx);

    loop {
        let (socket, peer) = listener.accept().await?;
        let tx = Arc::clone(&tx);
        let mut rx = tx.subscribe();
        tokio::spawn(async move {
            if let Err(e) = handle_client(socket, peer, tx, &mut rx).await {
                eprintln!("Client {} error: {:?}", peer, e);
            }
        });
    }
}

async fn handle_client(
    socket: TcpStream,
    peer: SocketAddr,
    tx: Arc<broadcast::Sender<String>>,
    rx: &mut broadcast::Receiver<String>,
) -> Result<()> {
    let (reader, mut writer) = socket.into_split();
    let mut reader = BufReader::new(reader);
    let mut line = String::new();

    // announce join
    let _ = tx.send(format!("[server] {} joined", peer));

    loop {
        tokio::select! {
            bytes = reader.read_line(&mut line) => {
                if bytes? == 0 { break; }
                let msg = format!("{}: {}", peer, line.trim_end());
                let _ = tx.send(msg);
                line.clear();
            }
            Ok(msg) = rx.recv() => {
                writer.write_all(msg.as_bytes()).await?;
                writer.write_all(b"\n").await?;
            }
        }
    }

    let _ = tx.send(format!("[server] {} left", peer));
    Ok(())
}

async fn run_client(addr: String, nick: String) -> Result<()> {
    let mut stream = TcpStream::connect(&addr).await?;
    println!("Connected to {}", addr);

    let (reader, mut writer) = stream.split();
    let mut server_reader = BufReader::new(reader);
    let mut server_line = String::new();

    // stdin task
    let mut input = tokio::task::spawn_blocking(|| io::stdin().lock().lines());

    // optional nick prefix
    let nick_prefix = if nick.trim().is_empty() { String::new() } else { format!("[{}] ", nick) };

    loop {
        tokio::select! {
            // read from server
            n = server_reader.read_line(&mut server_line) => {
                if n? == 0 { println!("Disconnected."); break; }
                print!("{}", server_line);
                server_line.clear();
            }
            // read from stdin using blocking iterator forwarded via spawn_blocking
            Some(Ok(line)) = &mut input => {
                let msg = format!("{}{}", nick_prefix, line);
                writer.write_all(msg.as_bytes()).await?;
                writer.write_all(b"\n").await?;
            }
        }
    }
    Ok(())
}
