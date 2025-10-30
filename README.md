Berikut file panduan lengkap siap dibagikan ke mahasiswa.

---

#Panduan Praktik Flutter: Aplikasi Chat Group (Supabase + Provider)

**Program Studi:** D3 Sistem Informasi — Politeknik Negeri Nusa Utara
**Mata Kuliah:** Pemrograman Mobile
**Dosen Pengampu:** Arifin Tindi
**Target:** Mahasiswa Angkatan 2024

---

## 🎯 Tujuan

Membangun aplikasi chat group menggunakan Flutter dan Supabase dengan fitur:

* Register dan Login (email, password, NIM, nama)
* Chat room tunggal (semua user gabung)
* Bubble chat kanan (pesan sendiri), kiri (pesan orang lain)
* Daftar anggota
* Logout

---

## 1. Persiapan Awal

### 1.1 Instalasi Flutter

Pastikan Flutter terpasang:

```bash
flutter --version
```

Minimal versi **3.5.x**.

Jika belum ada, ikuti panduan:
[https://docs.flutter.dev/get-started/install](https://docs.flutter.dev/get-started/install)

---

### 1.2 Membuat Project

```bash
flutter create chat_group_app
cd chat_group_app
```

---

### 1.3 Menjalankan Proyek

```bash
flutter run
```

Pastikan muncul aplikasi Flutter default (“Hello World”).

---

## 2. Setup Supabase

### 2.1 Tambahkan dependency

Buka `pubspec.yaml` → ubah bagian dependencies menjadi:

```yaml
dependencies:
  flutter:
    sdk: flutter
  provider: ^6.0.0
  supabase_flutter: ^1.4.0
  uuid: ^3.0.6
  intl: ^0.18.0
```

Simpan lalu jalankan:

```bash
flutter pub get
```

---

### 2.2 File `lib/constants.dart`

Isi dengan kredensial Supabase:

```dart
const String SUPABASE_URL = 'https://eazwidfgernzenwttgwf.supabase.co';
const String SUPABASE_ANON_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImVhendpZGZnZXJuemVud3R0Z3dmIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NjE3OTg3NTUsImV4cCI6MjA3NzM3NDc1NX0.TdlILGKX1uqz2bqyNxx5JihouZ-RpdzR2hhUtS95egI';
```

Ganti dengan URL dan key dari project Supabase Anda.

---

## 3. Setup Database Supabase

Masuk ke [https://supabase.com/dashboard](https://supabase.com/dashboard)
Pilih project → **SQL Editor** → tempel dan jalankan:

```sql
drop table if exists messages cascade;
drop table if exists profiles cascade;

create table profiles (
  id uuid primary key default gen_random_uuid(),
  user_id uuid unique references auth.users(id) on delete cascade,
  email text not null,
  nim text,
  name text,
  created_at timestamptz default now()
);

create table messages (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(user_id) on delete cascade,
  content text not null,
  created_at timestamptz default now()
);

create index on messages (created_at);
create index on profiles (created_at);
```

---

### 3.1 Matikan RLS

Buka **Table Editor** → klik `profiles` dan `messages` → tab **Security** → nonaktifkan **RLS**.

---

## 4. Struktur Folder Flutter

```
lib/
  constants.dart
  main.dart
  models/
    profile.dart
    message.dart
  services/
    supabase_service.dart
  providers/
    auth_provider.dart
    chat_provider.dart
  screens/
    login_screen.dart
    register_screen.dart
    chat_screen.dart
    members_screen.dart
  widgets/
    chat_bubble.dart
```

---

## 5. Kode Program Lengkap

### 5.1 `lib/services/supabase_service.dart`

```dart
import 'package:supabase_flutter/supabase_flutter.dart';
import '../constants.dart';

class SupabaseService {
  static final SupabaseService _instance = SupabaseService._internal();
  late final SupabaseClient supabase;

  factory SupabaseService() => _instance;
  SupabaseService._internal();

  Future<void> init() async {
    await Supabase.initialize(url: SUPABASE_URL, anonKey: SUPABASE_ANON_KEY);
    supabase = Supabase.instance.client;
  }
}
```

---

### 5.2 `lib/models/profile.dart`

```dart
class Profile {
  final String id;
  final String userId;
  final String email;
  final String nim;
  final String name;
  final DateTime createdAt;

  Profile({
    required this.id,
    required this.userId,
    required this.email,
    required this.nim,
    required this.name,
    required this.createdAt,
  });

  factory Profile.fromMap(Map<String, dynamic> m) => Profile(
        id: m['id'],
        userId: m['user_id'],
        email: m['email'],
        nim: m['nim'] ?? '',
        name: m['name'] ?? '',
        createdAt: DateTime.parse(m['created_at']),
      );
}
```

---

### 5.3 `lib/models/message.dart`

```dart
class ChatMessage {
  final String id;
  final String userId;
  final String content;
  final DateTime createdAt;

  ChatMessage({
    required this.id,
    required this.userId,
    required this.content,
    required this.createdAt,
  });

  factory ChatMessage.fromMap(Map<String, dynamic> m) => ChatMessage(
        id: m['id'],
        userId: m['user_id'],
        content: m['content'],
        createdAt: DateTime.parse(m['created_at']),
      );
}
```

---

### 5.4 `lib/providers/auth_provider.dart`

```dart
import 'package:flutter/material.dart';
import 'package:supabase_flutter/supabase_flutter.dart';
import '../services/supabase_service.dart';
import '../models/profile.dart';

class AuthProvider extends ChangeNotifier {
  final SupabaseClient _supabase = SupabaseService().supabase;
  Profile? profile;

  Future<String?> register({
    required String email,
    required String password,
    required String nim,
    required String name,
  }) async {
    final res = await _supabase.auth.signUp(email: email, password: password);
    if (res.user == null) return 'Registrasi gagal';

    await _supabase.from('profiles').insert({
      'user_id': res.user!.id,
      'email': email,
      'nim': nim,
      'name': name,
    });
    return null;
  }

  Future<String?> login(String email, String password) async {
    final res = await _supabase.auth.signInWithPassword(email: email, password: password);
    if (res.user == null) return 'Email atau password salah';
    return null;
  }

  Future<void> logout() async {
    await _supabase.auth.signOut();
    profile = null;
    notifyListeners();
  }

  Future<List<Profile>> getAllProfiles() async {
    final data = await _supabase.from('profiles').select().order('created_at');
    return (data as List).map((e) => Profile.fromMap(e)).toList();
  }
}
```

---

### 5.5 `lib/providers/chat_provider.dart`

```dart
import 'dart:async';
import 'package:flutter/material.dart';
import '../models/message.dart';
import '../services/supabase_service.dart';
import 'package:supabase_flutter/supabase_flutter.dart';

class ChatProvider extends ChangeNotifier {
  final SupabaseClient _supabase = SupabaseService().supabase;
  List<ChatMessage> messages = [];
  StreamSubscription? _sub;

  Future<void> fetchMessages() async {
    final res = await _supabase.from('messages').select().order('created_at');
    messages = (res as List).map((m) => ChatMessage.fromMap(m)).toList();
    notifyListeners();
  }

  void listenMessages() {
    _sub = _supabase.from('messages').stream(primaryKey: ['id']).order('created_at').listen((rows) {
      messages = rows.map((m) => ChatMessage.fromMap(m)).toList();
      notifyListeners();
    });
  }

  Future<void> sendMessage(String userId, String content) async {
    await _supabase.from('messages').insert({'user_id': userId, 'content': content});
  }

  @override
  void dispose() {
    _sub?.cancel();
    super.dispose();
  }
}
```

---

### 5.6 `lib/widgets/chat_bubble.dart`

```dart
import 'package:flutter/material.dart';
import '../models/message.dart';
import 'package:intl/intl.dart';

class ChatBubble extends StatelessWidget {
  final ChatMessage message;
  final bool isMe;
  const ChatBubble({required this.message, required this.isMe, Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    final time = DateFormat.Hm().format(message.createdAt.toLocal());
    return Align(
      alignment: isMe ? Alignment.centerRight : Alignment.centerLeft,
      child: Container(
        margin: EdgeInsets.all(8),
        padding: EdgeInsets.all(12),
        decoration: BoxDecoration(
          color: isMe ? Colors.blue : Colors.grey.shade300,
          borderRadius: BorderRadius.circular(12),
        ),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.end,
          children: [
            Text(message.content, style: TextStyle(color: isMe ? Colors.white : Colors.black)),
            SizedBox(height: 4),
            Text(time, style: TextStyle(fontSize: 10, color: isMe ? Colors.white70 : Colors.black54)),
          ],
        ),
      ),
    );
  }
}
```

---

### 5.7 `lib/screens/register_screen.dart`

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../providers/auth_provider.dart';

class RegisterScreen extends StatefulWidget {
  const RegisterScreen({Key? key}) : super(key: key);

  @override
  State<RegisterScreen> createState() => _RegisterScreenState();
}

class _RegisterScreenState extends State<RegisterScreen> {
  final emailCtrl = TextEditingController();
  final passwordCtrl = TextEditingController();
  final nimCtrl = TextEditingController();
  final nameCtrl = TextEditingController();
  bool loading = false;

  @override
  Widget build(BuildContext context) {
    final auth = Provider.of<AuthProvider>(context, listen: false);
    return Scaffold(
      appBar: AppBar(title: Text('Register')),
      body: Padding(
        padding: EdgeInsets.all(16),
        child: Column(children: [
          TextField(controller: emailCtrl, decoration: InputDecoration(labelText: 'Email')),
          TextField(controller: passwordCtrl, decoration: InputDecoration(labelText: 'Password'), obscureText: true),
          TextField(controller: nimCtrl, decoration: InputDecoration(labelText: 'NIM')),
          TextField(controller: nameCtrl, decoration: InputDecoration(labelText: 'Nama')),
          SizedBox(height: 12),
          ElevatedButton(
            onPressed: loading ? null : () async {
              setState(() => loading = true);
              final err = await auth.register(
                email: emailCtrl.text,
                password: passwordCtrl.text,
                nim: nimCtrl.text,
                name: nameCtrl.text,
              );
              setState(() => loading = false);
              if (err != null) {
                ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(err)));
              } else {
                Navigator.pushReplacementNamed(context, '/chat');
              }
            },
            child: loading ? CircularProgressIndicator() : Text('Register'),
          ),
          TextButton(
            onPressed: () => Navigator.pushReplacementNamed(context, '/login'),
            child: Text('Sudah punya akun? Login'),
          )
        ]),
      ),
    );
  }
}
```

---

### 5.8 `lib/screens/login_screen.dart`

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../providers/auth_provider.dart';

class LoginScreen extends StatefulWidget {
  const LoginScreen({Key? key}) : super(key: key);

  @override
  State<LoginScreen> createState() => _LoginScreenState();
}

class _LoginScreenState extends State<LoginScreen> {
  final emailCtrl = TextEditingController();
  final passwordCtrl = TextEditingController();
  bool loading = false;

  @override
  Widget build(BuildContext context) {
    final auth = Provider.of<AuthProvider>(context, listen: false);
    return Scaffold(
      appBar: AppBar(title: Text('Login')),
      body: Padding(
        padding: EdgeInsets.all(16),
        child: Column(children: [
          TextField(controller: emailCtrl, decoration: InputDecoration(labelText: 'Email')),
          TextField(controller: passwordCtrl, decoration: InputDecoration(labelText: 'Password'), obscureText: true),
          SizedBox(height: 12),
          ElevatedButton(
            onPressed: loading ? null : () async {
              setState(() => loading = true);
              final err = await auth.login(emailCtrl.text, passwordCtrl.text);
              setState(() => loading = false);
              if (err != null) {
                ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(err)));
              } else {
                Navigator.pushReplacementNamed(context, '/chat');
              }
            },
            child: loading ? CircularProgressIndicator() : Text('Login'),
          ),
          TextButton(
            onPressed: () => Navigator.pushReplacementNamed(context, '/register'),
            child: Text('Belum punya akun? Daftar'),
          )
        ]),
      ),
    );
  }
}
```

---

### 5.9 `lib/screens/chat_screen.dart`

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../providers/chat_provider.dart';
import '../providers/auth_provider.dart';
import '../widgets/chat_bubble.dart';

class ChatScreen extends StatefulWidget {
  const ChatScreen({Key? key}) : super(key: key);
  @override
  State<ChatScreen> createState() => _ChatScreenState();
}

class _ChatScreenState extends State<ChatScreen> {
  final msgCtrl = TextEditingController();

  @override
  void initState() {
    super.initState();
    final chat = Provider.of<ChatProvider>(context, listen: false);
    chat.fetchMessages();
    chat.listenMessages();
  }

  @override
  Widget build(BuildContext context) {
    final auth = Provider.of<AuthProvider>(context, listen: false);
    return Scaffold(
      appBar: AppBar(
        title: Text('Group Chat'),
        actions: [
          IconButton(
            icon: Icon(Icons.people),
            onPressed: () => Navigator.pushNamed(context, '/members'),
          ),
          IconButton(
            icon: Icon(Icons.logout),
            onPressed: () async {
              await auth.logout();
              Navigator.pushReplacementNamed(context, '/login');
            },
          ),
        ],
      ),
      body: Column(
        children: [
          Expanded(
            child: Consumer<ChatProvider>(
              builder: (context, chat, _) {
                return ListView.builder(
                  itemCount: chat.messages.length,
                  itemBuilder: (context, i) {
                    final m = chat.messages[i];
                    final isMe = m.userId == auth._supabase.auth.currentUser?.id;
                    return ChatBubble(message: m, isMe: isMe);
                  },
                );
              },
            ),
          ),
          SafeArea(
            child: Row(children: [
              Expanded(child: TextField(controller: msgCtrl, decoration: InputDecoration(hintText: 'Tulis pesan...'))),
              IconButton(
                icon: Icon(Icons.send),
                onPressed: () async {
                  if (msgCtrl.text.isEmpty) return;
                  await Provider.of<ChatProvider>(context, listen: false)
                      .sendMessage(auth._supabase.auth.currentUser!.id, msgCtrl.text);
                  msgCtrl.clear();
                },
              )
            ]),
          ),
        ],
      ),
    );
  }
}
```

---

### 5.10 `lib/screens/members_screen.dart`

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../providers/auth_provider.dart';
import '../models/profile.dart';

class MembersScreen extends StatefulWidget {
  const MembersScreen({Key? key}) : super(key: key);

  @override
  State<MembersScreen> createState() => _MembersScreenState();
}

class _MembersScreenState extends State<MembersScreen> {
  List<Profile> members = [];
  bool loading = true;

  @override
  void initState() {
    super.initState();
    loadMembers();
  }

  Future<void> loadMembers() async {
    final auth = Provider.of<AuthProvider>(context, listen: false);
    final data = await auth.getAllProfiles();
    setState(() {
      members = data;
      loading = false;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Daftar Anggota')),
      body: loading
          ? Center(child: CircularProgressIndicator())
          : ListView.builder(
              itemCount: members.length,
              itemBuilder: (_, i) => ListTile(
                title: Text(members[i].name),
                subtitle: Text('${members[i].nim} • ${members[i].email}'),
              ),
            ),
    );
  }
}
```

---

### 5.11 `lib/main.dart`

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'services/supabase_service.dart';
import 'providers/auth_provider.dart';
import 'providers/chat_provider.dart';
import 'screens/login_screen.dart';
import 'screens/register_screen.dart';
import 'screens/chat_screen.dart';
import 'screens/members_screen.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await SupabaseService().init();
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MultiProvider(
      providers: [
        ChangeNotifierProvider(create: (_) => AuthProvider()),
        ChangeNotifierProvider(create: (_) => ChatProvider()),
      ],
      child: MaterialApp(
        title: 'Group Chat',
        theme: ThemeData(primarySwatch: Colors.blue),
        initialRoute: '/login',
        routes: {
          '/login': (_) => LoginScreen(),
          '/register': (_) => RegisterScreen(),
          '/chat': (_) => ChatScreen(),
          '/members': (_) => MembersScreen(),
        },
      ),
    );
  }
}
```

---

## 6. Menjalankan Aplikasi

```bash
flutter run
```

Register → Login → Chat Room.
Coba di dua perangkat untuk melihat chat realtime.

---

## 7. Rubrik Penilaian

| No | Fitur              | Keterangan                  | Nilai |
| -- | ------------------ | --------------------------- | ----- |
| 1  | Register & Login   | Autentikasi Supabase        | 20    |
| 2  | Chat Room          | Bubble kanan-kiri           | 30    |
| 3  | Daftar Anggota     | Menampilkan semua user      | 20    |
| 4  | Logout             | Keluar dan kembali ke login | 10    |
| 5  | UI & Struktur Kode |                             |       |
