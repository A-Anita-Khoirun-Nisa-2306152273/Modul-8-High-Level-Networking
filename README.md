# Modul-8-High-Level-Networking
## 1. What are the key differences between unary, server streaming, and bi-directional streaming RPC (Remote Procedure Call) methods, and in what scenarios would each be most suitable?

| Jenis | Pola Komunikasi | Jumlah Request | Jumlah Response |
|-------|----------------|----------------|-----------------|
| **Unary** | Client → Server | 1 | 1 |
| **Server Streaming** | Client → Server → Stream | 1 | Banyak |
| **Bidirectional Streaming** | Client ↔ Server | Banyak | Banyak |

**Kapan digunakan:**
- **Unary**: Login, submit form, payment processing, request data tunggal
- **Server Streaming**: Get transaction history, real-time notification, download file besar
- **Bidirectional Streaming**: Chat application, live collaboration, IoT telemetry

---

## 2. What are the potential security considerations involved in implementing a gRPC service in Rust, particularly regarding authentication, authorization, and data encryption?

**A. Authentication (Siapa pengguna?)**
- Gunakan JWT atau API key
- Taruh token di metadata gRPC
- Implementasikan interceptor untuk validasi setiap request

**B. Authorization (Apa yang boleh dilakukan?)**
- Role-Based Access Control (RBAC)
- Cek permission di awal setiap method handler

**C. Data Encryption**
- WAJIB menggunakan TLS/SSL di production
- Rust + tonic mendukung TLS via rustls atau native-tls
- Jangan kirim data sensitif dalam plain text

**D. Best Practice**
```rust
Server::builder()
    .tls_config(load_tls_config())?
    .add_service(MyService::new())
    .serve(addr)
    .await?;
```

---

## 3. What are the potential challenges or issues that may arise when handling bidirectional streaming in Rust gRPC, especially in scenarios like chat applications?

| Tantangan | Penyebab | Solusi |
|-----------|----------|--------|
| Message ordering | Pesan tidak berurutan | Tambahkan sequence number atau timestamp |
| Concurrent access | Banyak user bersamaan | Gunakan tokio::sync::Mutex atau RwLock |
| Memory leak | Stream tidak ditutup | Implementasi timeout dengan tokio::time::timeout |
| Backpressure | Server kewalahan | Pakai bounded channel: mpsc::channel(4) |
| Connection drop | Client disconnect | Handle error di stream.next() dan break loop |
| Resource exhaustion | Terlalu banyak koneksi | Batasi jumlah concurrent connections |

**Contoh implementasi aman:**
```rust
loop {
    match tokio::time::timeout(Duration::from_secs(30), stream.next()).await {
        Ok(Some(Ok(msg))) => process_message(msg).await,
        Ok(Some(Err(e))) => break,
        Ok(None) => break,
        Err(_) => break,
    }
}
```

---

## 4. What are the advantages and disadvantages of using the tokio_stream::wrappers::ReceiverStream for streaming responses in Rust gRPC services?

**Kelebihan (+):**
- Mudah digunakan - konversi langsung dari Receiver ke Stream
- Terintegrasi dengan tokio - support async/await
- Komposabel - bisa di-map(), filter(), take()
- Cocok untuk prototyping - cepat dan minim boilerplate

**Kekurangan (-):**
- Receiver akan closed jika semua sender di-drop
- Overhead performa - lebih lambat dari custom Stream
- Sekali pakai - stream hanya bisa dikonsumsi sekali
- Error handling terbatas - sulit untuk custom error logic
- Tidak reusable - tidak bisa dipakai ulang setelah stream habis

**Kesimpulan:** Cocok untuk prototyping. Untuk production dengan kebutuhan kompleks, implementasikan custom Stream.

---

## 5. In what ways could the Rust gRPC code be structured to facilitate code reuse and modularity,promoting maintainability and extensibility over time?

**Struktur folder yang direkomendasikan:**
```
src/
├── proto/              # Generated code dari protobuf
│   └── mod.rs
├── server/
│   ├── mod.rs          # Inisialisasi server
│   ├── payment.rs      # PaymentService implementation
│   ├── transaction.rs  # TransactionService implementation
│   └── chat.rs         # ChatService implementation
├── client/
│   ├── mod.rs          # Inisialisasi client
│   └── handlers.rs     # Client request handlers
├── common/
│   ├── mod.rs          # Shared types & utilities
│   ├── errors.rs       # Custom error types
│   ├── auth.rs         # Authentication logic
│   └── config.rs       # Configuration management
└── main.rs             # Entry point
```

**Prinsip modularitas:**
- Single Responsibility - Satu file/struct untuk satu service
- Dependency Injection - Gunakan trait untuk memisahkan implementasi
- Shared logic - Taruh di common module untuk reuse
- Configuration - Gunakan Config struct dengan environment variables
- Error handling - Custom error type dengan thiserror atau anyhow

---

## 6. In the MyPaymentService implementation, what additional steps might be necessary to handle more complex payment processing logic?

**Komponen yang perlu ditambahkan untuk production:**
- **Database**: sqlx atau diesel untuk PostgreSQL/MySQL
- **Validation**: validator crate untuk validasi input
- **Error handling**: Custom error types dengan thiserror
- **Idempotency**: Cek duplicate request dengan transaction_id
- **Logging**: tracing untuk structured logging
- **Metrics**: prometheus + opentelemetry
- **Retry logic**: backoff crate untuk retry dengan exponential backoff
- **Timeout**: tokio::time::timeout per request
- **Rate limiting**: governor crate untuk limit request
- **Circuit breaker**: Custom implementasi atau tower middleware

**Contoh implementasi:**
```rust
#[tonic::async_trait]
impl PaymentService for MyPaymentService {
    async fn process_payment(
        &self,
        request: Request<PaymentRequest>,
    ) -> Result<Response<PaymentResponse>, Status> {
        let req = request.into_inner();
        
        // 1. Validate input
        if req.amount <= 0.0 {
            return Err(Status::invalid_argument("Amount must be positive"));
        }
        
        // 2. Check idempotency
        if self.is_duplicate(&req.transaction_id).await? {
            return Ok(Response::new(PaymentResponse { success: true }));
        }
        
        // 3. Process with timeout
        match tokio::time::timeout(Duration::from_secs(10), self.process(req)).await {
            Ok(Ok(resp)) => Ok(Response::new(resp)),
            Ok(Err(e)) => Err(Status::internal(e.to_string())),
            Err(_) => Err(Status::deadline_exceeded("Payment timeout")),
        }
    }
}
```

---

## 7. What impact does the adoption of gRPC as a communication protocol have on the overall architecture and design of distributed systems, particularly in terms of interoperability with other technologies and platforms?


**Dampak Positif (+):**
- Interoperability: Bisa digunakan dengan bahasa lain (Go, Java, Python, C#)
- Performance: Lebih cepat dari REST karena binary serialization
- Code generation: Client/server otomatis dari .proto file
- Type safety: Compile-time checking untuk API contract
- Streaming native: Support bi-directional streaming out-of-the-box

**Dampak Negatif (-):**
- Kurang human-readable (binary, tidak seperti JSON)
- Browser support terbatas (perlu gRPC-web)
- Learning curve lebih tinggi (protobuf, streaming)
- Debugging lebih susah (perlu tools seperti grpcurl)
- Tidak cocok untuk public API yang perlu diakses browser langsung

---

## 8. What are the advantages and disadvantages of using HTTP/2, the underlying protocol for gRPC, compared to HTTP/1.1 or HTTP/1.1 with WebSocket for REST APIs?

| Aspek | HTTP/2 (gRPC) | HTTP/1.1 (REST) |
|-------|---------------|-----------------|
| Multiplexing | Banyak request dalam 1 koneksi | Perlu multiple koneksi |
| Header compression | HPACK (efisien) | Plain text |
| Server push | Bisa push data tanpa diminta | Tidak ada |
| Binary vs text | Binary (Protobuf) - kecil | Text (JSON) - besar |
| Browser support | Terbatas (gRPC-web) | Universal |
| Payload size | Kecil | Besar |
| Human readable | Tidak | Ya |
| Streaming | Native (bi-directional) | Hanya via WebSocket |
| Latency | Rendah (1 koneksi) | Tinggi (handshake tiap request) |

**Kesimpulan:**
- Gunakan gRPC untuk internal service-to-service communication (microservices)
- Gunakan REST untuk public API yang perlu diakses browser

---

## 9. How does the request-response model of REST APIs contrast with the bidirectional streaming capabilities of gRPC in terms of real-time communication and responsiveness?

| Aspek | REST (HTTP/1.1) | gRPC Bidirectional Stream |
|-------|-----------------|---------------------------|
| Pola komunikasi | Client request → Server response (sekali) | Client dan server bisa kirim kapan saja |
| Real-time | Tidak (perlu pooling/WebSocket) | Ya |
| Latency | Tinggi (setiap request handshake) | Rendah (satu koneksi) |
| Server push | Tidak ada | Bisa push data tanpa diminta |
| Stateful vs Stateless | Stateless | Stateful (perlu maintain connection) |
| Overhead | Besar (header + JSON) | Kecil (binary + header compression) |
| Use case | CRUD, form submission | Chat, gaming, live tracking, IoT |

**Contoh konkret:**
- REST Chat: Polling setiap 5 detik → latency 5 detik
- gRPC Chat: Via bidirectional stream → real-time (<100ms)

---

## 10. What are the implications of the schema-based approach of gRPC, using Protocol Buffers, compared to the more flexible, schema-less nature of JSON in REST API payloads?

| Aspek | Protobuf (gRPC) | JSON (REST) |
|-------|-----------------|-------------|
| Ukuran payload | Kecil (binary) | Besar (text) |
| Kecepatan parsing | Cepat | Lambat |
| Type safety | Compile-time (deteksi error saat compile) | Runtime (error baru ketahuan saat jalan) |
| Backward compatibility | Supported (via field tags) | Manual (perlu hati-hati) |
| Human readable | Binary → perlu tools khusus | Mudah debug langsung |
| Flexibility | Rigid (harus define schema dulu) | Bebas tambah field kapan saja |
| Documentation | .proto file sebagai source of truth | OpenAPI/Swagger (terpisah) |
| Code generation | Otomatis dari .proto | Manual atau tool terpisah |
| Breaking changes | Mudah detect karena type mismatch | Susah detect, perlu testing |

**Rekomendasi:**
- Gunakan Protobuf untuk: Internal API, microservices, mobile apps
- Gunakan JSON untuk: Public API, prototyping, browser-based apps