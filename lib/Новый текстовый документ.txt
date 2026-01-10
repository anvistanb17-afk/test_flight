import 'package:flutter/material.dart';
import 'package:supabase_flutter/supabase_flutter.dart';
import 'package:provider/provider.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:sqflite/sqflite.dart';
import 'package:sqflite_common_ffi/sqflite_ffi.dart';
import 'package:path/path.dart' as p;
import 'package:image_picker/image_picker.dart';
import 'package:intl/intl.dart';
import 'dart:math';
import 'dart:ui';
import 'dart:io';
import 'dart:convert';
import 'dart:async';

// --- БД СЕРВИС (SQLite) ---
class DBHelper {
  static Database? _db;
  static Future<Database> get database async {
    if (_db != null) return _db!;
    if (Platform.isWindows || Platform.isLinux) {
      sqfliteFfiInit();
      databaseFactory = databaseFactoryFfi;
    }
    _db = await openDatabase(
      p.join(await getDatabasesPath(), 'chat_v31_ultimate.db'),
      onCreate: (db, version) => db.execute(
        'CREATE TABLE messages(id TEXT PRIMARY KEY, content TEXT, username TEXT, user_id TEXT, room_id TEXT, created_at TEXT, is_image INTEGER DEFAULT 0)',
      ),
      version: 1,
    );
    return _db!;
  }

  static Future<void> saveLocal(Map<String, dynamic> m) async {
    final db = await database;
    await db.insert(
        'messages',
        {
          'id': m['id'].toString(),
          'content': m['content'],
          'username': m['username'],
          'user_id': m['user_id'],
          'room_id': m['room_id'],
          'created_at': m['created_at'],
          'is_image': (m['is_image'] == true || m['is_image'] == 1) ? 1 : 0,
        },
        conflictAlgorithm: ConflictAlgorithm.replace);
  }

  static Future<void> deleteLocalMessage(String id) async {
    final db = await database;
    await db.delete('messages', where: 'id = ?', whereArgs: [id]);
  }

  static Future<List<Map<String, dynamic>>> getLocal(String roomId) async {
    final db = await database;
    return await db.query('messages',
        where: 'room_id LIKE ?',
        whereArgs: ['$roomId%'],
        orderBy: 'created_at DESC');
  }

  static Future<void> clearLocalOnly() async =>
      (await (await database).delete('messages'));
}

// --- ЦЕНТРАЛЬНЫЙ КОНФИГ ---
class AppConfig with ChangeNotifier {
  String nickname = "User";
  String userId = "";
  String? pinCode;
  String? avatarUrl;
  bool isOnline = true;
  int bgIndex = 0;
  Color themeColor = Colors.cyan;
  double fontSize = 16.0;
  double bubbleRadius = 18.0;
  Map<String, String> roomPasswords = {};
  Map<String, String> profileCache = {};
  Map<String, String?> avatarCache = {};

  final List<List<Color>> backgrounds = [
    [const Color(0xFF0F2027), const Color(0xFF203A43)],
    [const Color(0xFF1D2671), const Color(0xFFC33764)],
    [const Color(0xFF000000), const Color(0xFF434343)],
  ];

  final List<Color> accentColors = [
    Colors.cyan,
    Colors.greenAccent,
    Colors.orangeAccent,
    Colors.pinkAccent,
    Colors.deepPurpleAccent,
    Colors.blueAccent,
    Colors.redAccent,
  ];

  AppConfig() {
    _load();
  }

  void _load() async {
    final prefs = await SharedPreferences.getInstance();
    userId = prefs.getString('u_id_v2') ?? _genId();
    await prefs.setString('u_id_v2', userId);
    nickname = prefs.getString('nick') ?? "User_${Random().nextInt(100)}";
    avatarUrl = prefs.getString('user_avatar');
    pinCode = prefs.getString('app_pin');
    bgIndex = prefs.getInt('bg') ?? 0;
    fontSize = prefs.getDouble('font') ?? 16.0;
    bubbleRadius = prefs.getDouble('radius') ?? 18.0;
    themeColor = Color(prefs.getInt('color') ?? Colors.cyan.value);

    final List<String> savedRooms = prefs.getStringList('rooms_list') ?? [];
    for (var r in savedRooms) {
      roomPasswords[r] = prefs.getString('pass_$r') ?? "";
    }
    notifyListeners();
    _syncProfile();
  }

  String _genId() => List.generate(15,
          (i) => 'abcdefghijklmnopqrstuvwxyz0123456789'[Random().nextInt(36)])
      .join();

  void setOnline(bool v) {
    if (isOnline != v) {
      isOnline = v;
      notifyListeners();
    }
  }

  Future<void> _syncProfile() async {
    try {
      await Supabase.instance.client
          .from('users_profiles')
          .upsert({'user_id': userId, 'nickname': nickname, 'avatar_url': avatarUrl});
    } catch (_) {}
  }

  Future<Map<String, String?>> getLiveUserData(String uid) async {
    if (uid == userId) return {'nick': nickname, 'avatar': avatarUrl};
    if (profileCache.containsKey(uid)) {
      return {'nick': profileCache[uid], 'avatar': avatarCache[uid]};
    }
    try {
      final res = await Supabase.instance.client
          .from('users_profiles')
          .select('nickname, avatar_url')
          .eq('user_id', uid)
          .maybeSingle();
      if (res != null) {
        profileCache[uid] = res['nickname'];
        avatarCache[uid] = res['avatar_url'];
        return {'nick': res['nickname'], 'avatar': res['avatar_url']};
      }
    } catch (_) {}
    return {'nick': "Anon", 'avatar': null};
  }

  void updateNickname(String v) {
    nickname = v;
    SharedPreferences.getInstance().then((p) => p.setString('nick', v));
    _syncProfile();
    notifyListeners();
  }

  void updateAvatar(String base64) {
    avatarUrl = base64;
    SharedPreferences.getInstance().then((p) => p.setString('user_avatar', base64));
    _syncProfile();
    notifyListeners();
  }

  void setBg(int i) {
    bgIndex = i;
    SharedPreferences.getInstance().then((p) => p.setInt('bg', i));
    notifyListeners();
  }

  void setThemeColor(Color c) {
    themeColor = c;
    SharedPreferences.getInstance().then((p) => p.setInt('color', c.value));
    notifyListeners();
  }

  void setFontSize(double v) {
    fontSize = v;
    SharedPreferences.getInstance().then((p) => p.setDouble('font', v));
    notifyListeners();
  }

  void setRadius(double v) {
    bubbleRadius = v;
    SharedPreferences.getInstance().then((p) => p.setDouble('radius', v));
    notifyListeners();
  }

  void setPin(String? p) {
    pinCode = p;
    SharedPreferences.getInstance().then(
        (pr) => p == null ? pr.remove('app_pin') : pr.setString('app_pin', p));
    notifyListeners();
  }

  void addRoom(String n, String p) {
    roomPasswords[n] = p;
    SharedPreferences.getInstance().then((pr) {
      pr.setString('pass_$n', p);
      pr.setStringList('rooms_list', roomPasswords.keys.toList());
    });
    notifyListeners();
  }

  void removeRoom(String n) {
    roomPasswords.remove(n);
    SharedPreferences.getInstance().then((pr) {
      pr.remove('pass_$n');
      pr.setStringList('rooms_list', roomPasswords.keys.toList());
    });
    notifyListeners();
  }
}

// --- ЭКРАН ПУСТОТЫ ---
class EmptyState extends StatefulWidget {
  final String text;
  const EmptyState({super.key, required this.text});
  @override
  State<EmptyState> createState() => _EmptyStateState();
}

class _EmptyStateState extends State<EmptyState>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  final String emoji = ["👻", "👾", "🤖", "🛸", "🐱"][Random().nextInt(5)];
  @override
  void initState() {
    super.initState();
    _controller =
        AnimationController(vsync: this, duration: const Duration(seconds: 2))
          ..repeat(reverse: true);
  }

  @override
  Widget build(BuildContext context) {
    return Center(
        child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
      ScaleTransition(
          scale: Tween(begin: 0.8, end: 1.2).animate(_controller),
          child: Text(emoji, style: const TextStyle(fontSize: 60))),
      Text(widget.text, style: const TextStyle(color: Colors.white38))
    ]));
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}

// --- ПРОСМОТР ФОТО ---
class FullImagePage extends StatelessWidget {
  final String imgBase64;
  const FullImagePage({super.key, required this.imgBase64});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.black,
      appBar: AppBar(backgroundColor: Colors.transparent, iconTheme: const IconThemeData(color: Colors.white)),
      body: Center(
        child: InteractiveViewer(
          child: Image.memory(base64Decode(imgBase64)),
        ),
      ),
    );
  }
}

// --- MAIN ---
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Supabase.initialize(
      url: 'https://pwkwpiznnuoyktzgrcce.supabase.co',
      anonKey: 'sb_publishable_F-Ba1Eo1lzmmWdQxS4EGnA_qSBQcEwi');

  final prefs = await SharedPreferences.getInstance();
  final bool isFirstRun = prefs.getBool('is_first_run_v31') ?? true;

  runApp(ChangeNotifierProvider(
      create: (_) => AppConfig(), 
      child: ChatApp(isFirstRun: isFirstRun)
  ));
}

class ChatApp extends StatelessWidget {
  final bool isFirstRun;
  const ChatApp({super.key, required this.isFirstRun});

  @override
  Widget build(BuildContext context) {
    final conf = Provider.of<AppConfig>(context);
    Widget homeWidget;
    if (isFirstRun) {
      homeWidget = const GlitchScreen();
    } else {
      homeWidget = (conf.pinCode == null)
          ? const ChatListScreen()
          : const PinLockScreen();
    }

    return MaterialApp(
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
          useMaterial3: true,
          brightness: Brightness.dark,
          colorSchemeSeed: conf.themeColor),
      home: homeWidget,
    );
  }
}

class GlitchScreen extends StatefulWidget {
  const GlitchScreen({super.key});
  @override
  State<GlitchScreen> createState() => _GlitchScreenState();
}

class _GlitchScreenState extends State<GlitchScreen> {
  bool _isBlack = true;
  @override
  void initState() {
    super.initState();
    _simulateError();
  }
  void _simulateError() async {
    await Future.delayed(const Duration(seconds: 3)); // Уменьшил для теста
    final prefs = await SharedPreferences.getInstance();
    await prefs.setBool('is_first_run_v31', false);
    if (!mounted) return;
    setState(() => _isBlack = false);
    await Future.delayed(const Duration(seconds: 1));
    if (!mounted) return;
    final conf = Provider.of<AppConfig>(context, listen: false);
    Navigator.pushReplacement(context, MaterialPageRoute(builder: (_) => (conf.pinCode == null) ? const ChatListScreen() : const PinLockScreen()));
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.black,
      body: Center(
        child: _isBlack 
          ? const SizedBox.shrink() 
          : const Text("CRITICAL_ERROR: RECOVERING...", style: TextStyle(color: Colors.greenAccent, fontSize: 12, fontFamily: 'monospace')),
      ),
    );
  }
}

class PinLockScreen extends StatefulWidget {
  const PinLockScreen({super.key});
  @override
  State<PinLockScreen> createState() => _PinLockScreenState();
}

class _PinLockScreenState extends State<PinLockScreen> {
  final _c = TextEditingController();
  @override
  Widget build(BuildContext context) {
    final conf = Provider.of<AppConfig>(context);
    return Scaffold(
      body: Container(
          decoration: BoxDecoration(gradient: LinearGradient(colors: conf.backgrounds[conf.bgIndex])),
          child: Center(
              child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
                const Icon(Icons.lock_outline, size: 50, color: Colors.white24),
                SizedBox(width: 150, child: TextField(controller: _c, obscureText: true, textAlign: TextAlign.center, style: const TextStyle(fontSize: 30), decoration: const InputDecoration(hintText: "PIN"), onChanged: (v) {
                  if (v == conf.pinCode) Navigator.pushReplacement(context, MaterialPageRoute(builder: (_) => const ChatListScreen()));
                })),
              ]))),
    );
  }
}

class ChatListScreen extends StatelessWidget {
  const ChatListScreen({super.key});
  @override
  Widget build(BuildContext context) {
    final conf = Provider.of<AppConfig>(context);
    return Scaffold(
      appBar: AppBar(title: const Text("Ваши комнаты"), actions: [
        IconButton(icon: const Icon(Icons.settings), onPressed: () => Navigator.push(context, MaterialPageRoute(builder: (_) => const SettingsPage())))
      ]),
      body: Container(
          decoration: BoxDecoration(gradient: LinearGradient(colors: conf.backgrounds[conf.bgIndex])),
          child: conf.roomPasswords.isEmpty
              ? const EmptyState(text: "Создайте первую комнату")
              : ListView.builder(
                  itemCount: conf.roomPasswords.length,
                  itemBuilder: (c, i) {
                    final room = conf.roomPasswords.keys.elementAt(i);
                    return ListTile(
                        leading: CircleAvatar(backgroundColor: conf.themeColor, child: const Icon(Icons.chat_bubble_outline, color: Colors.white)),
                        title: Text(room),
                        trailing: IconButton(icon: const Icon(Icons.delete_sweep), onPressed: () => conf.removeRoom(room)),
                        onTap: () => Navigator.push(context, MaterialPageRoute(builder: (_) => ChatPage(roomId: room))));
                  })),
      floatingActionButton: FloatingActionButton(onPressed: () => _add(context, conf), child: const Icon(Icons.add)),
    );
  }

  void _add(BuildContext ctx, AppConfig conf) {
    final n = TextEditingController(), p = TextEditingController();
    showDialog(context: ctx, builder: (c) => AlertDialog(title: const Text("Вход"), content: Column(mainAxisSize: MainAxisSize.min, children: [
      TextField(controller: n, decoration: const InputDecoration(labelText: "Комната")),
      TextField(controller: p, decoration: const InputDecoration(labelText: "Пароль"))
    ]), actions: [ElevatedButton(onPressed: () { if (n.text.isNotEmpty) { conf.addRoom(n.text, p.text); Navigator.pop(c); } }, child: const Text("ОК"))]));
  }
}

class ChatPage extends StatefulWidget {
  final String roomId;
  const ChatPage({super.key, required this.roomId});
  @override
  State<ChatPage> createState() => _ChatPageState();
}

class _ChatPageState extends State<ChatPage> {
  final _ctrl = TextEditingController();
  List<Map<String, dynamic>> _displayMsgs = [];
  StreamSubscription? _sub;

  @override
  void initState() {
    super.initState();
    _startLogic();
  }

  Future<void> _startLogic() async {
    final cached = await DBHelper.getLocal(widget.roomId);
    if (mounted) setState(() => _displayMsgs = cached);
    final conf = Provider.of<AppConfig>(context, listen: false);
    final rKey = "${widget.roomId}_${conf.roomPasswords[widget.roomId]}";
    _sub = Supabase.instance.client.from('messages').stream(primaryKey: ['id']).eq('room_id', rKey).order('created_at', ascending: false).listen((data) {
          for (var m in data) { DBHelper.saveLocal(m); }
          if (mounted) { setState(() => _displayMsgs = data); conf.setOnline(true); }
        }, onError: (e) { if (mounted) conf.setOnline(false); });
  }

  void _send(AppConfig conf, {String? img}) async {
    if (_ctrl.text.isEmpty && img == null) return;
    final rKey = "${widget.roomId}_${conf.roomPasswords[widget.roomId]}";
    final content = img ?? _ctrl.text;
    _ctrl.clear();
    try {
      await Supabase.instance.client.from('messages').insert({
        'content': content,
        'username': conf.nickname,
        'user_id': conf.userId,
        'room_id': rKey,
        'is_image': img != null,
        'created_at': DateTime.now().toIso8601String(),
      });
    } catch (_) { conf.setOnline(false); }
  }

  @override
  void dispose() { _sub?.cancel(); _ctrl.dispose(); super.dispose(); }

  @override
  Widget build(BuildContext context) {
    final conf = Provider.of<AppConfig>(context);
    return Scaffold(
      extendBodyBehindAppBar: true,
      appBar: AppBar(title: Text(conf.isOnline ? widget.roomId : "${widget.roomId} (Оффлайн)"), backgroundColor: Colors.black26, flexibleSpace: ClipRect(child: BackdropFilter(filter: ImageFilter.blur(sigmaX: 5, sigmaY: 5), child: Container(color: Colors.transparent)))),
      body: Container(
          decoration: BoxDecoration(gradient: LinearGradient(colors: conf.backgrounds[conf.bgIndex])),
          child: Column(children: [
            Expanded(child: _displayMsgs.isEmpty ? const EmptyState(text: "Тишина...") : ListView.builder(reverse: true, padding: const EdgeInsets.only(top: 100), itemCount: _displayMsgs.length, itemBuilder: (c, i) => _Bubble(m: _displayMsgs[i], conf: conf))),
            _input(conf),
          ])),
    );
  }

  Widget _input(AppConfig conf) {
    return SafeArea(child: Padding(padding: const EdgeInsets.all(8), child: Row(children: [
      IconButton(icon: const Icon(Icons.image), onPressed: () async {
        final img = await ImagePicker().pickImage(source: ImageSource.gallery, imageQuality: 20);
        if (img != null) { final bytes = await img.readAsBytes(); _send(conf, img: base64Encode(bytes)); }
      }),
      Expanded(child: TextField(controller: _ctrl, decoration: const InputDecoration(hintText: "Пиши..."))),
      IconButton(icon: Icon(Icons.send, color: conf.themeColor), onPressed: () => _send(conf)),
    ])));
  }
}

class _Bubble extends StatelessWidget {
  final Map<String, dynamic> m;
  final AppConfig conf;
  const _Bubble({required this.m, required this.conf});

  void _showMenu(BuildContext context) {
    showDialog(context: context, builder: (ctx) => AlertDialog(
      title: const Text("Удалить сообщение?"),
      actions: [
        TextButton(onPressed: () => Navigator.pop(ctx), child: const Text("Отмена")),
        TextButton(onPressed: () async {
          await Supabase.instance.client.from('messages').delete().eq('id', m['id']);
          await DBHelper.deleteLocalMessage(m['id'].toString());
          if (context.mounted) Navigator.pop(ctx);
        }, child: const Text("Удалить", style: TextStyle(color: Colors.red))),
      ],
    ));
  }

  @override
  Widget build(BuildContext context) {
    final isMe = m['user_id'] == conf.userId;
    String timeStr = "";
    if (m['created_at'] != null) {
      try { timeStr = DateFormat('HH:mm').format(DateTime.parse(m['created_at']).toLocal()); } catch (_) {}
    }

    return FutureBuilder<Map<String, String?>>(
      future: conf.getLiveUserData(m['user_id']),
      builder: (c, snap) {
        final userData = snap.data;
        return GestureDetector(
          onLongPress: isMe ? () => _showMenu(context) : null,
          child: Align(
            alignment: isMe ? Alignment.centerRight : Alignment.centerLeft,
            child: Container(
              margin: const EdgeInsets.symmetric(vertical: 5, horizontal: 10),
              padding: const EdgeInsets.all(12),
              constraints: BoxConstraints(maxWidth: MediaQuery.of(context).size.width * 0.75),
              decoration: BoxDecoration(
                  color: isMe ? conf.themeColor.withOpacity(0.8) : Colors.white10,
                  borderRadius: BorderRadius.only(
                    topLeft: Radius.circular(conf.bubbleRadius),
                    topRight: Radius.circular(conf.bubbleRadius),
                    bottomLeft: Radius.circular(isMe ? conf.bubbleRadius : 0),
                    bottomRight: Radius.circular(isMe ? 0 : conf.bubbleRadius),
                  )),
              child: Column(crossAxisAlignment: isMe ? CrossAxisAlignment.end : CrossAxisAlignment.start, children: [
                Row(mainAxisSize: MainAxisSize.min, children: [
                  CircleAvatar(radius: 8, backgroundImage: userData?['avatar'] != null ? MemoryImage(base64Decode(userData!['avatar']!)) : null, child: userData?['avatar'] == null ? const Icon(Icons.person, size: 10) : null),
                  const SizedBox(width: 5),
                  Text(userData?['nick'] ?? "...", style: const TextStyle(fontSize: 10, color: Colors.white54)),
                  const SizedBox(width: 6),
                  Text(timeStr, style: const TextStyle(fontSize: 10, color: Colors.white24)),
                ]),
                const SizedBox(height: 4),
                m['is_image'] == 1 || m['is_image'] == true
                    ? GestureDetector(
                        onTap: () => Navigator.push(context, MaterialPageRoute(builder: (_) => FullImagePage(imgBase64: m['content']))),
                        child: ClipRRect(borderRadius: BorderRadius.circular(8), child: Image.memory(base64Decode(m['content']), errorBuilder: (context, error, stackTrace) => const Icon(Icons.broken_image))))
                    : Text(m['content'] ?? "", style: TextStyle(fontSize: conf.fontSize, color: Colors.white)),
              ]),
            ),
          ),
        );
      },
    );
  }
}

class SettingsPage extends StatelessWidget {
  const SettingsPage({super.key});
  @override
  Widget build(BuildContext context) {
    final conf = Provider.of<AppConfig>(context);
    return Scaffold(
      appBar: AppBar(title: const Text("Настройки")),
      body: ListView(padding: const EdgeInsets.all(20), children: [
        const Text("Ваш аватар:", style: TextStyle(fontWeight: FontWeight.bold)),
        const SizedBox(height: 10),
        Center(child: GestureDetector(
          onTap: () async {
            final img = await ImagePicker().pickImage(source: ImageSource.gallery, imageQuality: 15);
            if (img != null) { final bytes = await img.readAsBytes(); conf.updateAvatar(base64Encode(bytes)); }
          },
          child: CircleAvatar(radius: 40, backgroundColor: conf.themeColor, backgroundImage: conf.avatarUrl != null ? MemoryImage(base64Decode(conf.avatarUrl!)) : null, child: conf.avatarUrl == null ? const Icon(Icons.add_a_photo, size: 30) : null),
        )),
        const SizedBox(height: 20),
        TextField(decoration: InputDecoration(labelText: "Ник", hintText: conf.nickname), onSubmitted: (v) => conf.updateNickname(v)),
        const Divider(height: 40),
        const Text("Цвет акцента:", style: TextStyle(fontWeight: FontWeight.bold)),
        const SizedBox(height: 10),
        SizedBox(height: 50, child: ListView.builder(scrollDirection: Axis.horizontal, itemCount: conf.accentColors.length, itemBuilder: (context, i) {
              final color = conf.accentColors[i];
              return GestureDetector(onTap: () => conf.setThemeColor(color), child: Container(width: 45, margin: const EdgeInsets.only(right: 10), decoration: BoxDecoration(color: color, shape: BoxShape.circle, border: conf.themeColor.value == color.value ? Border.all(color: Colors.white, width: 3) : null)));
            })),
        const Divider(height: 40),
        const Text("Скругление сообщений:", style: TextStyle(fontWeight: FontWeight.bold)),
        Slider(value: conf.bubbleRadius, min: 0, max: 30, activeColor: conf.themeColor, onChanged: (v) => conf.setRadius(v)),
        const Divider(height: 40),
        const Text("Шрифт:", style: TextStyle(fontWeight: FontWeight.bold)),
        Slider(value: conf.fontSize, min: 12, max: 24, activeColor: conf.themeColor, onChanged: (v) => conf.setFontSize(v)),
        const Divider(height: 40),
        const Text("Фон приложения:", style: TextStyle(fontWeight: FontWeight.bold)),
        const SizedBox(height: 10),
        Row(children: [0, 1, 2].map((i) => GestureDetector(onTap: () => conf.setBg(i), child: Container(width: 50, height: 50, margin: const EdgeInsets.all(5), decoration: BoxDecoration(borderRadius: BorderRadius.circular(10), border: conf.bgIndex == i ? Border.all(color: Colors.white, width: 2) : null, gradient: LinearGradient(colors: conf.backgrounds[i]))))).toList()),
        const Divider(height: 40),
        ListTile(title: const Text("Установить PIN"), trailing: Icon(Icons.lock, color: conf.themeColor), onTap: () {
              final c = TextEditingController();
              showDialog(context: context, builder: (ctx) => AlertDialog(title: const Text("Новый PIN"), content: TextField(controller: c, keyboardType: TextInputType.number), actions: [
                            TextButton(onPressed: () { conf.setPin(null); Navigator.pop(ctx); }, child: const Text("Сброс")),
                            ElevatedButton(onPressed: () { conf.setPin(c.text); Navigator.pop(ctx); }, child: const Text("ОК"))
                          ]));
            }),
        const Divider(),
        ListTile(leading: const Icon(Icons.delete_forever, color: Colors.redAccent), title: const Text("Очистить кэш сообщений", style: TextStyle(color: Colors.redAccent)), onTap: () async {
              await DBHelper.clearLocalOnly();
              if (context.mounted) Navigator.pop(context);
            }),
      ]),
    );
  }
}