# 📘 Dev Guide – dx\_vas

Tài liệu này cung cấp hướng dẫn phát triển hệ thống dx\_vas cho toàn bộ đội ngũ kỹ thuật. Bao gồm cách cài đặt môi trường, quy ước code, cấu trúc dịch vụ, quản lý RBAC, CI/CD, và các quy trình vận hành liên quan.

---

## Mục lục

1. [Giới thiệu tổng quan](#1-giới-thiệu-tổng-quan)
2. [Cài đặt môi trường phát triển](#2-cài-đặt-môi-trường-phát-triển)
3. [Cấu trúc thư mục & dịch vụ](#3-cấu-trúc-thư-mục--dịch-vụ)
4. [Quy ước viết mã & Coding Style](#4-quy-ước-viết-mã--coding-style)
5. [RBAC – Phân quyền động](#5-rbac--phân-quyền-động)
6. [Thiết kế API & OpenAPI](#6-thiết-kế-api--openapi)
7. [Gửi thông báo (Notification)](#7-gửi-thông-báo-notification)
8. [Quy trình test & CI/CD](#8-quy-trình-test--ci-cd)
9. [Migration & Quản lý cơ sở dữ liệu](#9-migration--quản-lý-cơ-sở-dữ-liệu)
10. [Quy trình pull request & review](#10-quy-trình-pull-request--review)
11. [Theo dõi & vận hành](#11-theo-dõi--vận-hành)
12. [Troubleshooting – Lỗi thường gặp](#12-troubleshooting--lỗi-thường-gặp)

---

## 1. Giới thiệu tổng quan

* Hệ thống dx\_vas bao gồm các thành phần:

  * API Gateway (FastAPI)
  * Core Services: Auth Service, User Service, Notification Service
  * Business Adapters: CRM Adapter, SIS Adapter, LMS Adapter
  * Các thành phần hạ tầng: Redis, PostgreSQL, MySQL, Pub/Sub, GCS
* Kiến trúc microservices, tất cả triển khai trên Google Cloud Run
* Mọi truy cập đều thông qua API Gateway với kiểm soát RBAC động
* Sơ đồ hệ thống đầy đủ tại: [`docs/system-diagrams.md`](../system-diagrams.md)

## 2. Cài đặt môi trường phát triển

(TBD)

## 3. Cấu trúc thư mục & dịch vụ

(TBD)

## 4. Quy ước viết mã & Coding Style

(TBD)

## 5. RBAC – Phân quyền động

(TBD)

## 6. Thiết kế API & OpenAPI

(TBD)

## 7. Gửi thông báo (Notification)

(TBD)

## 8. Quy trình test & CI/CD

(TBD)

## 9. Migration & Quản lý cơ sở dữ liệu

(TBD)

## 10. Quy trình pull request & review

(TBD)

## 11. Theo dõi & vận hành

(TBD)

## 12. Troubleshooting – Lỗi thường gặp

(TBD)
