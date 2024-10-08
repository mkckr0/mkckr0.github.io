---
title: asio 使用 openssl 示例
date: 2022-05-28 00:12
---

```cpp
#include <boost/asio.hpp>
#include <boost/asio/ssl.hpp>
#include <boost/algorithm/string.hpp>
#include <boost/lexical_cast.hpp>
#define OPENSSL_NO_DEPRECATED
#include <openssl/ssl.h>
#include <wincrypt.h>
#pragma comment(lib, "Crypt32.lib")

#include <iostream>
#include <fstream>
#include <string_view>

void fail(const char* what, const boost::system::error_code& ec) {
    std::cout << what << ": " << ec.message() << std::endl;
}

namespace asio = boost::asio;
using tcp = asio::ip::tcp;

int add_ca(asio::ssl::context& ssl_ctx, boost::system::error_code& ec)
{
    BIO* bio = BIO_new_fp(stdout, BIO_NOCLOSE);
    if (!bio) {
        return -1;
    }

    auto cert_store = CertOpenSystemStore(NULL, L"ROOT");
    if (!cert_store) {
        return -1;
    }

    const unsigned char* encoded_cert = nullptr;
    int i = 0;
    for (PCCERT_CONTEXT cert = nullptr; cert = CertEnumCertificatesInStore(cert_store, cert);) {
        X509* x = d2i_X509(NULL, (const unsigned char**)&cert->pbCertEncoded, cert->cbCertEncoded);
        if (x) {
            BIO* bio_mem = BIO_new(BIO_s_mem());
            PEM_write_bio_X509(bio_mem, x);
            const char* data = nullptr;
            auto len = BIO_get_mem_data(bio_mem, &data);

            ssl_ctx.add_certificate_authority(asio::buffer(data, len), ec);

            BIO_free(bio_mem);

            if (ec) {
                return -1;
            }
        }
        ++i;
    }

    BIO_printf(bio, "cert sum: %d\n", i);

    CertCloseStore(cert_store, CERT_CLOSE_STORE_FORCE_FLAG);

    return 0;
}

int main()
{
    boost::system::error_code ec;

    asio::io_context io_ctx;

    tcp::resolver resolver(io_ctx);
    auto results = resolver.resolve("www.baidu.com", "https", ec);
    if (ec) {
        fail("resolver.resolve", ec);
        return -1;
    }

    asio::ssl::context ssl_ctx(asio::ssl::context::method::sslv23_client);
    add_ca(ssl_ctx, ec);
    if (ec) {
        fail("add_ca", ec);
        return -1;
    }

    asio::ssl::stream<tcp::socket> s(io_ctx, ssl_ctx);
    s.set_verify_mode(asio::ssl::verify_peer | asio::ssl::verify_client_once);
    asio::connect(s.lowest_layer(), results, ec);
    if (ec) {
        fail("asio::connect", ec);
        return -1;
    }

    s.handshake(s.client, ec);
    if (ec) {
        fail("s.handshake", ec);
        return -1;
    }

    const char* req = "GET / HTTP/1.1\r\nHost: www.baidu.com\r\nAccept: text/html\r\n\r\n";
    asio::write(s, asio::buffer(req, std::strlen(req)), ec);
    if (ec) {
        fail("asio::write", ec);
        return -1;
    }

    // Header
    asio::streambuf streambuf;
    std::size_t content_length = 0;
    constexpr std::string_view delimiter = "\r\n";
    constexpr std::string_view header_name = "Content-Length";
    while (true) {
        std::size_t len = asio::read_until(s, streambuf, delimiter, ec);
        if (ec) {
            fail("asio::read_until", ec);
            return -1;
        }

        std::string_view header(asio::buffer_cast<const char*>(streambuf.data()), len - delimiter.size());

        std::cout << header << std::endl;

        if (!header.empty()) {
            if (boost::algorithm::istarts_with(header, header_name)) {
                bool has_optional_space = header[header_name.size() + 1] == ' ';
                std::string_view header_value(header.cbegin() + header_name.size() + 1 + has_optional_space, header.cend());
                content_length = boost::lexical_cast<std::size_t>(header_value);
            }
        }

        streambuf.consume(len);

        if (header.empty()) {
            break;
        }
    }

    // Body
    std::cout << "content_length: " << content_length << std::endl;
    
    std::size_t len = asio::read(s, streambuf.prepare(content_length - streambuf.size()), ec);
    if (ec) {
        fail("asio::read", ec);
        return -1;
    }
    streambuf.commit(len);

    std::string_view body(asio::buffer_cast<const char*>(streambuf.data()), streambuf.size());

    std::system("chcp 65001");
    std::cout << body;

    streambuf.consume(streambuf.size());

    s.shutdown(ec);
    if (ec) {
        fail("s.shutdown", ec);
        return -1;
    }

    s.lowest_layer().shutdown(tcp::socket::shutdown_both, ec);
    if (ec) {
        fail("s.lowest_layer().shutdown", ec);
        return -1;
    }

    return 0;
}
```
