¡Qué tal! Estructurar el desarrollo de **FinanTrack** utilizando una arquitectura basada en agentes (`.agents`) es una estrategia excelente para automatizar y escalar el proyecto. Vamos a establecer la metodología, configurar el entorno y generar el código base centralizado en operaciones CRUD, asegurando que cada bloque de código esté completo y listo para usarse.

Aquí tienes el plan de acción estructurado.

---

### Fase 1: Arquitectura de Agentes y Flujo de Trabajo

Primero, creamos la estructura global de automatización que guiará el desarrollo. En la raíz de tu entorno de trabajo, esta será la estructura de la carpeta `.agents`:

```text
.agents/
├── SKILL.md                 # Definición de roles, metodologías y directrices
├── scripts/
│   ├── setup_env.sh         # Script para validar instalaciones (Flutter, Firebase CLI)
│   └── deploy_rules.sh      # Script para actualizar reglas de seguridad en Firestore
├── ejemplos/
│   ├── auth_flow_mock.dart  # Código de referencia para el UI Agent
│   └── crud_pattern.dart    # Patrón estándar para el Database Agent
└── resources/
    ├── UI_palette.json      # Colores y tipografías para FinanTrack
    └── db_schema.json       # Estructura JSON de las colecciones de Firestore
```

**`SKILL.md` (Definición del Ecosistema de Agentes):**
```markdown
# FinanTrack - Agent Skills & Workflow

## Roles Activos
1. **Design_Agent (UI/UX):** Encargado de construir las pantallas en `lib/screens/` siguiendo los lineamientos de Material Design. Sus entregables deben incluir widgets modulares y navegación por rutas nombradas.
2. **Code_Agent (Lógica/Routing):** Responsable de la integración de estados, validaciones de formularios y el enrutamiento en `main.dart`.
3. **Database_Agent (Backend/Firebase):** Encargado de las operaciones en `lib/services/` y `lib/models/`. Maneja Firebase Auth y Cloud Firestore, asegurando la relación de datos mediante `id_usuario`.

## Flujo de Trabajo (Workflow)
1. **Paso 1 (DB):** Crear modelos de datos completos con métodos de serialización (`toMap`, `fromMap`).
2. **Paso 2 (Code/DB):** Configurar servicios de Firebase y métodos CRUD.
3. **Paso 3 (Design):** Construir la UI conectando los servicios mediante controladores de estado.
```

---

### Fase 2: Prerrequisitos y Preparación del Entorno

Abre tu terminal en VS Code o IDE Antigravity y ejecuta los siguientes comandos para preparar la conexión con Firebase:

1. **Crear el proyecto Flutter:**
   ```bash
   flutter create crudfinanzas
   cd crudfinanzas
   ```
2. **Autenticarse e inicializar Firebase:**
   ```bash
   firebase login
   dart pub global activate flutterfire_cli
   flutterfire configure
   ```
   *(Selecciona tu proyecto "FinanTrack" en la consola de Firebase y marca Android e iOS).*
3. **Agregar dependencias:**
   ```bash
   flutter pub add firebase_core cloud_firestore firebase_auth
   ```

---

### Fase 3: Estructura del Proyecto Flutter (`crudfinanzas/lib`)

Organiza tu carpeta `lib` de la siguiente manera para mantener el código limpio y modular:

```text
lib/
├── models/
│   ├── gasto.dart
│   ├── ingreso.dart
│   └── usuario.dart
├── services/
│   └── firestore_service.dart
├── screens/
│   ├── home_screen.dart
│   ├── login_screen.dart
│   └── ... (resto de pantallas)
└── main.dart
```

---

### Fase 4: Modelos de Datos (Database_Agent)

Estos modelos relacionan la información mediante el `id_usuario` para que cada persona vea solo sus propias finanzas.

**`lib/models/gasto.dart`**
```dart
class Gasto {
  String id;
  String idUsuario;
  double monto;
  String categoria;
  String descripcion;
  DateTime fecha;

  Gasto({
    required this.id,
    required this.idUsuario,
    required this.monto,
    required this.categoria,
    required this.descripcion,
    required this.fecha,
  });

  factory Gasto.fromMap(Map<String, dynamic> data, String documentId) {
    return Gasto(
      id: documentId,
      idUsuario: data['id_usuario'] ?? '',
      monto: (data['monto'] ?? 0.0).toDouble(),
      categoria: data['categoria'] ?? '',
      descripcion: data['descripcion'] ?? '',
      fecha: data['fecha'] != null ? data['fecha'].toDate() : DateTime.now(),
    );
  }

  Map<String, dynamic> toMap() {
    return {
      'id_usuario': idUsuario,
      'monto': monto,
      'categoria': categoria,
      'descripcion': descripcion,
      'fecha': fecha,
    };
  }
}
```

**`lib/models/ingreso.dart`**
```dart
class Ingreso {
  String id;
  String idUsuario;
  double monto;
  String fuente;
  DateTime fecha;

  Ingreso({
    required this.id,
    required this.idUsuario,
    required this.monto,
    required this.fuente,
    required this.fecha,
  });

  factory Ingreso.fromMap(Map<String, dynamic> data, String documentId) {
    return Ingreso(
      id: documentId,
      idUsuario: data['id_usuario'] ?? '',
      monto: (data['monto'] ?? 0.0).toDouble(),
      fuente: data['fuente'] ?? '',
      fecha: data['fecha'] != null ? data['fecha'].toDate() : DateTime.now(),
    );
  }

  Map<String, dynamic> toMap() {
    return {
      'id_usuario': idUsuario,
      'monto': monto,
      'fuente': fuente,
      'fecha': fecha,
    };
  }
}
```

---

### Fase 5: Servicios CRUD y Autenticación (Code_Agent & Database_Agent)

Este archivo centraliza la comunicación con Firestore y maneja el flujo de creación, lectura, actualización y borrado.

**`lib/services/firestore_service.dart`**
```dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:firebase_auth/firebase_auth.dart';
import '../models/gasto.dart';
import '../models/ingreso.dart';

class FirestoreService {
  final FirebaseFirestore _db = FirebaseFirestore.instance;
  final FirebaseAuth _auth = FirebaseAuth.instance;

  // Obtener ID del usuario actual autenticado
  String? get _userId => _auth.currentUser?.uid;

  // ==========================================
  // CRUD GASTOS
  // ==========================================

  // Crear Gasto
  Future<void> addGasto(Gasto gasto) async {
    if (_userId == null) return;
    gasto.idUsuario = _userId!;
    await _db.collection('gastos').add(gasto.toMap());
  }

  // Leer Gastos (Stream para tiempo real)
  Stream<List<Gasto>> getGastos() {
    if (_userId == null) return Stream.value([]);
    return _db
        .collection('gastos')
        .where('id_usuario', isEqualTo: _userId)
        .orderBy('fecha', descending: true)
        .snapshots()
        .map((snapshot) => snapshot.docs
            .map((doc) => Gasto.fromMap(doc.data(), doc.id))
            .toList());
  }

  // Actualizar Gasto
  Future<void> updateGasto(Gasto gasto) async {
    await _db.collection('gastos').doc(gasto.id).update(gasto.toMap());
  }

  // Borrar Gasto
  Future<void> deleteGasto(String idGasto) async {
    await _db.collection('gastos').doc(idGasto).delete();
  }

  // ==========================================
  // CRUD INGRESOS
  // ==========================================

  // Crear Ingreso
  Future<void> addIngreso(Ingreso ingreso) async {
    if (_userId == null) return;
    ingreso.idUsuario = _userId!;
    await _db.collection('ingresos').add(ingreso.toMap());
  }

  // Leer Ingresos (Stream)
  Stream<List<Ingreso>> getIngresos() {
    if (_userId == null) return Stream.value([]);
    return _db
        .collection('ingresos')
        .where('id_usuario', isEqualTo: _userId)
        .orderBy('fecha', descending: true)
        .snapshots()
        .map((snapshot) => snapshot.docs
            .map((doc) => Ingreso.fromMap(doc.data(), doc.id))
            .toList());
  }

  // Actualizar Ingreso
  Future<void> updateIngreso(Ingreso ingreso) async {
    await _db.collection('ingresos').doc(ingreso.id).update(ingreso.toMap());
  }

  // Borrar Ingreso
  Future<void> deleteIngreso(String idIngreso) async {
    await _db.collection('ingresos').doc(idIngreso).delete();
  }
}
```

---

### Fase 6: Configuración Principal y Enrutamiento (Code_Agent)

Para navegar eficientemente, configuramos `main.dart` con rutas nombradas. 

**`lib/main.dart`**
```dart
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'firebase_options.dart'; // Generado automáticamente por flutterfire_cli

// Importa aquí tus pantallas una vez que el Design_Agent las genere
// import 'screens/login_screen.dart';
// import 'screens/home_screen.dart';
// import 'screens/resumen_mensual.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );
  runApp(const FinanTrackApp());
}

class FinanTrackApp extends StatelessWidget {
  const FinanTrackApp({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'FinanTrack',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        primarySwatch: Colors.blue,
        useMaterial3: true,
      ),
      // Definición de rutas nombradas
      initialRoute: '/login', // Cambia a '/' si el usuario ya está logueado
      routes: {
        // '/login': (context) => const LoginScreen(),
        // '/registro': (context) => const RegistroScreen(),
        // '/home': (context) => const HomeScreen(),
        // '/captura_gasto': (context) => const CapturaGastoScreen(),
        // '/captura_ingreso': (context) => const CapturaIngresoScreen(),
        // '/resumen': (context) => const ResumenMensualScreen(),
        // '/perfil': (context) => const PerfilScreen(),
      },
      // Pantalla temporal de carga mientras se construyen las rutas
      home: const Scaffold(
        body: Center(
          child: CircularProgressIndicator(),
        ),
      ),
    );
  }
}
```

Hemos establecido exitosamente el entorno, la arquitectura de automatización y el backend en código. El siguiente paso en nuestro flujo de trabajo involucra al **Design_Agent** para construir las interfaces. 

¿Te gustaría que comencemos desarrollando la pantalla de `Login/Registro` con Firebase Auth, o prefieres que pasemos directamente a las pantallas de `Captura de Gastos/Ingresos` para probar el CRUD?
