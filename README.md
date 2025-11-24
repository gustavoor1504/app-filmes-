# app-filmes-
import 'dart:async';
import 'dart:io'; // Para Directory (se necessário em algumas plataformas)
import 'package:flutter/material.dart';
import 'package:flutter_rating_bar/flutter_rating_bar.dart';
import 'package:path/path.dart';
import 'package:sqflite/sqflite.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Gerenciador de Filmes',
      theme: ThemeData(
        primarySwatch: Colors.deepPurple,
        useMaterial3: true,
      ),
      home: const MovieListScreen(),
      debugShowCheckedModeBanner: false,
    );
  }
}

// ==========================================
// MODEL (M do MVC)
// ==========================================

class Movie {
  int? id;
  String title;
  String imageUrl;
  String genre;
  String ageRating; // Faixa etária
  String duration;
  double score; // Pontuação
  String description;
  int year;

  Movie({
    this.id,
    required this.title,
    required this.imageUrl,
    required this.genre,
    required this.ageRating,
    required this.duration,
    required this.score,
    required this.description,
    required this.year,
  });

  // Converte objeto para Map (para o SQLite)
  Map<String, dynamic> toMap() {
    return {
      'id': id,
      'title': title,
      'imageUrl': imageUrl,
      'genre': genre,
      'ageRating': ageRating,
      'duration': duration,
      'score': score,
      'description': description,
      'year': year,
    };
  }

  // Converte Map para objeto (do SQLite)
  factory Movie.fromMap(Map<String, dynamic> map) {
    return Movie(
      id: map['id'],
      title: map['title'],
      imageUrl: map['imageUrl'],
      genre: map['genre'],
      ageRating: map['ageRating'],
      duration: map['duration'],
      score: map['score'],
      description: map['description'],
      year: map['year'],
    );
  }
}

// ==========================================
// CONTROLLER / DATABASE HELPER
// ==========================================

class DatabaseHelper {
  static final DatabaseHelper instance = DatabaseHelper._init();
  static Database? _database;

  DatabaseHelper._init();

  Future<Database> get database async {
    if (_database != null) return _database!;
    _database = await _initDB('movies.db');
    return _database!;
  }

  Future<Database> _initDB(String filePath) async {
    final dbPath = await getDatabasesPath();
    final path = join(dbPath, filePath);

    return await openDatabase(path, version: 1, onCreate: _createDB);
  }

  Future _createDB(Database db, int version) async {
    const idType = 'INTEGER PRIMARY KEY AUTOINCREMENT';
    const textType = 'TEXT NOT NULL';
    const doubleType = 'REAL NOT NULL';
    const intType = 'INTEGER NOT NULL';

    await db.execute('''
      CREATE TABLE movies (
        id $idType,
        title $textType,
        imageUrl $textType,
        genre $textType,
        ageRating $textType,
        duration $textType,
        score $doubleType,
        description $textType,
        year $intType
      )
    ''');
  }

  Future<int> create(Movie movie) async {
    final db = await instance.database;
    return await db.insert('movies', movie.toMap());
  }

  Future<Movie> readMovie(int id) async {
    final db = await instance.database;
    final maps = await db.query(
      'movies',
      columns: ['id', 'title', 'imageUrl', 'genre', 'ageRating', 'duration', 'score', 'description', 'year'],
      where: 'id = ?',
      whereArgs: [id],
    );

    if (maps.isNotEmpty) {
      return Movie.fromMap(maps.first);
    } else {
      throw Exception('ID $id not found');
    }
  }

  Future<List<Movie>> readAllMovies() async {
    final db = await instance.database;
    final result = await db.query('movies', orderBy: 'title ASC');
    return result.map((json) => Movie.fromMap(json)).toList();
  }

  Future<int> update(Movie movie) async {
    final db = await instance.database;
    return db.update(
      'movies',
      movie.toMap(),
      where: 'id = ?',
      whereArgs: [movie.id],
    );
  }

  Future<int> delete(int id) async {
    final db = await instance.database;
    return await db.delete(
      'movies',
      where: 'id = ?',
      whereArgs: [id],
    );
  }
}

// ==========================================
// VIEWS (Telas)
// ==========================================

// 1. TELA LISTAR FILMES
class MovieListScreen extends StatefulWidget {
  const MovieListScreen({super.key});

  @override
  State<MovieListScreen> createState() => _MovieListScreenState();
}

class _MovieListScreenState extends State<MovieListScreen> {
  late Future<List<Movie>> movies;

  @override
  void initState() {
    super.initState();
    refreshMovies();
  }

  Future refreshMovies() async {
    setState(() {
      movies = DatabaseHelper.instance.readAllMovies();
    });
  }

  void _showGroupAlert() {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Integrantes do Grupo'),
        content: const Text('Nome 1\nNome 2\nNome 3'),
        actions: [
          TextButton(onPressed: () => Navigator.pop(context), child: const Text('OK'))
        ],
      ),
    );
  }

  void _showOptions(BuildContext context, Movie movie) {
    showModalBottomSheet(
      context: context,
      builder: (context) {
        return Wrap(
          children: [
            ListTile(
              leading: const Icon(Icons.visibility),
              title: const Text('Exibir Dados'),
              onTap: () {
                Navigator.pop(context);
                Navigator.push(
                  context,
                  MaterialPageRoute(
                    builder: (context) => MovieDetailsScreen(movie: movie),
                  ),
                );
              },
            ),
            ListTile(
              leading: const Icon(Icons.edit),
              title: const Text('Alterar'),
              onTap: () async {
                Navigator.pop(context);
                await Navigator.push(
                  context,
                  MaterialPageRoute(
                    builder: (context) => MovieFormScreen(movie: movie),
                  ),
                );
                refreshMovies();
              },
            ),
          ],
        );
      },
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Meus Filmes'),
        backgroundColor: Colors.deepPurple.shade100,
        actions: [
          IconButton(
            icon: const Icon(Icons.info_outline),
            onPressed: _showGroupAlert,
            tooltip: 'Ver Grupo',
          )
        ],
      ),
      body: FutureBuilder<List<Movie>>(
        future: movies,
        builder: (context, snapshot) {
          if (snapshot.connectionState == ConnectionState.waiting) {
            return const Center(child: CircularProgressIndicator());
          } else if (snapshot.hasError) {
            return Center(child: Text('Erro: ${snapshot.error}'));
          } else if (!snapshot.hasData || snapshot.data!.isEmpty) {
            return const Center(child: Text('Nenhum filme cadastrado.'));
          }

          return ListView.builder(
            itemCount: snapshot.data!.length,
            itemBuilder: (context, index) {
              final movie = snapshot.data![index];
              
              // Implementação do Dismissible para deletar arrastando
              return Dismissible(
                key: Key(movie.id.toString()),
                direction: DismissDirection.endToStart,
                background: Container(
                  color: Colors.red,
                  alignment: Alignment.centerRight,
                  padding: const EdgeInsets.symmetric(horizontal: 20),
                  child: const Icon(Icons.delete, color: Colors.white),
                ),
                onDismissed: (direction) async {
                  await DatabaseHelper.instance.delete(movie.id!);
                  ScaffoldMessenger.of(context).showSnackBar(
                    SnackBar(content: Text('${movie.title} deletado')),
                  );
                },
                child: Card(
                  margin: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
                  child: ListTile(
                    leading: SizedBox(
                      width: 50,
                      height: 50,
                      child: Image.network(
                        movie.imageUrl,
                        fit: BoxFit.cover,
                        errorBuilder: (c, o, s) => const Icon(Icons.error),
                      ),
                    ),
                    title: Text(movie.title),
                    subtitle: RatingBarIndicator(
                      rating: movie.score,
                      itemBuilder: (context, index) => const Icon(
                        Icons.star,
                        color: Colors.amber,
                      ),
                      itemCount: 5,
                      itemSize: 20.0,
                      direction: Axis.horizontal,
                    ),
                    onTap: () => _showOptions(context, movie),
                  ),
                ),
              );
            },
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        child: const Icon(Icons.add),
        onPressed: () async {
          await Navigator.push(
            context,
            MaterialPageRoute(builder: (context) => const MovieFormScreen()),
          );
          refreshMovies();
        },
      ),
    );
  }
}

// 2. TELA FORMULÁRIO (CADASTRAR / ALTERAR)
class MovieFormScreen extends StatefulWidget {
  final Movie? movie;

  const MovieFormScreen({super.key, this.movie});

  @override
  State<MovieFormScreen> createState() => _MovieFormScreenState();
}

class _MovieFormScreenState extends State<MovieFormScreen> {
  final _formKey = GlobalKey<FormState>();
  
  late TextEditingController _titleController;
  late TextEditingController _imageController;
  late TextEditingController _genreController;
  late TextEditingController _durationController;
  late TextEditingController _descController;
  late TextEditingController _yearController;
  
  String? _selectedAgeRating;
  double _currentScore = 0.0;

  final List<String> _ageRatings = ['Livre', '10', '12', '14', '16', '18'];

  @override
  void initState() {
    super.initState();
    _titleController = TextEditingController(text: widget.movie?.title ?? '');
    _imageController = TextEditingController(text: widget.movie?.imageUrl ?? '');
    _genreController = TextEditingController(text: widget.movie?.genre ?? '');
    _durationController = TextEditingController(text: widget.movie?.duration ?? '');
    _descController = TextEditingController(text: widget.movie?.description ?? '');
    _yearController = TextEditingController(text: widget.movie?.year.toString() ?? '');
    
    _selectedAgeRating = widget.movie?.ageRating;
    _currentScore = widget.movie?.score ?? 0.0;
  }

  @override
  void dispose() {
    _titleController.dispose();
    _imageController.dispose();
    _genreController.dispose();
    _durationController.dispose();
    _descController.dispose();
    _yearController.dispose();
    super.dispose();
  }

  Future<void> _saveMovie() async {
    if (_formKey.currentState!.validate()) {
      if (_selectedAgeRating == null) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Selecione uma faixa etária')),
        );
        return;
      }

      final movie = Movie(
        id: widget.movie?.id,
        title: _titleController.text,
        imageUrl: _imageController.text,
        genre: _genreController.text,
        ageRating: _selectedAgeRating!,
        duration: _durationController.text,
        score: _currentScore,
        description: _descController.text,
        year: int.tryParse(_yearController.text) ?? 0,
      );

      if (widget.movie == null) {
        await DatabaseHelper.instance.create(movie);
      } else {
        await DatabaseHelper.instance.update(movie);
      }

      if (mounted) Navigator.pop(context);
    }
  }

  @override
  Widget build(BuildContext context) {
    final isEditing = widget.movie != null;
    
    return Scaffold(
      appBar: AppBar(
        title: Text(isEditing ? 'Alterar Filme' : 'Cadastrar Filme'),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Form(
          key: _formKey,
          child: ListView(
            children: [
              TextFormField(
                controller: _titleController,
                decoration: const InputDecoration(labelText: 'Título'),
                validator: (value) => value!.isEmpty ? 'Campo obrigatório' : null,
              ),
              TextFormField(
                controller: _imageController,
                decoration: const InputDecoration(labelText: 'URL da Imagem'),
                validator: (value) => value!.isEmpty ? 'Campo obrigatório' : null,
              ),
              TextFormField(
                controller: _genreController,
                decoration: const InputDecoration(labelText: 'Gênero'),
                validator: (value) => value!.isEmpty ? 'Campo obrigatório' : null,
              ),
              TextFormField(
                controller: _yearController,
                decoration: const InputDecoration(labelText: 'Ano'),
                keyboardType: TextInputType.number,
                validator: (value) => value!.isEmpty ? 'Campo obrigatório' : null,
              ),
              TextFormField(
                controller: _durationController,
                decoration: const InputDecoration(labelText: 'Duração (ex: 2h 30m)'),
                validator: (value) => value!.isEmpty ? 'Campo obrigatório' : null,
              ),
              const SizedBox(height: 16),
              DropdownButtonFormField<String>(
                value: _selectedAgeRating,
                decoration: const InputDecoration(labelText: 'Faixa Etária'),
                items: _ageRatings.map((String age) {
                  return DropdownMenuItem<String>(
                    value: age,
                    child: Text(age),
                  );
                }).toList(),
                onChanged: (value) {
                  setState(() {
                    _selectedAgeRating = value;
                  });
                },
                validator: (value) => value == null ? 'Selecione a faixa etária' : null,
              ),
              const SizedBox(height: 16),
              const Text('Pontuação:'),
              RatingBar.builder(
                initialRating: _currentScore,
                minRating: 0,
                direction: Axis.horizontal,
                allowHalfRating: true,
                itemCount: 5,
                itemPadding: const EdgeInsets.symmetric(horizontal: 4.0),
                itemBuilder: (context, _) => const Icon(
                  Icons.star,
                  color: Colors.amber,
                ),
                onRatingUpdate: (rating) {
                  setState(() {
                    _currentScore = rating;
                  });
                },
              ),
              const SizedBox(height: 16),
              TextFormField(
                controller: _descController,
                decoration: const InputDecoration(
                  labelText: 'Descrição',
                  border: OutlineInputBorder(),
                ),
                maxLines: 4, // Requisito: usar maxLines para descrição
                validator: (value) => value!.isEmpty ? 'Campo obrigatório' : null,
              ),
              const SizedBox(height: 24),
              ElevatedButton(
                onPressed: _saveMovie,
                style: ElevatedButton.styleFrom(
                  padding: const EdgeInsets.symmetric(vertical: 16),
                ),
                child: Text(isEditing ? 'Salvar Alterações' : 'Cadastrar'),
              ),
            ],
          ),
        ),
      ),
    );
  }
}

// 3. TELA DE DETALHES DO FILME
class MovieDetailsScreen extends StatelessWidget {
  final Movie movie;

  const MovieDetailsScreen({super.key, required this.movie});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Detalhes do Filme'),
      ),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16.0),
        child: Row(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Layout com Imagem à esquerda
            SizedBox(
              width: 140,
              child: AspectRatio(
                aspectRatio: 0.7,
                child: ClipRRect(
                  borderRadius: BorderRadius.circular(8),
                  child: Image.network(
                    movie.imageUrl,
                    fit: BoxFit.cover,
                    errorBuilder: (c, o, s) => Container(
                      color: Colors.grey[300],
                      child: const Icon(Icons.broken_image, size: 50),
                    ),
                  ),
                ),
              ),
            ),
            const SizedBox(width: 16),
            // Informações à direita
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(
                    movie.title,
                    style: const TextStyle(
                      fontSize: 24,
                      fontWeight: FontWeight.bold,
                    ),
                  ),
                  const SizedBox(height: 8),
                  Text('Gênero: ${movie.genre}'),
                  Text('Ano: ${movie.year}'),
                  Text('Duração: ${movie.duration}'),
                  const SizedBox(height: 8),
                  Row(
                    children: [
                      Container(
                        padding: const EdgeInsets.symmetric(
                          horizontal: 8,
                          vertical: 4,
                        ),
                        decoration: BoxDecoration(
                          color: Colors.grey[800],
                          borderRadius: BorderRadius.circular(4),
                        ),
                        child: Text(
                          movie.ageRating,
                          style: const TextStyle(
                            color: Colors.white,
                            fontWeight: FontWeight.bold,
                          ),
                        ),
                      ),
                      const SizedBox(width: 10),
                      RatingBarIndicator(
                        rating: movie.score,
                        itemBuilder: (context, index) => const Icon(
                          Icons.star,
                          color: Colors.amber,
                        ),
                        itemCount: 5,
                        itemSize: 20.0,
                        direction: Axis.horizontal,
                      ),
                    ],
                  ),
                  const SizedBox(height: 16),
                  const Text(
                    'Sinopse:',
                    style: TextStyle(fontWeight: FontWeight.bold, fontSize: 16),
                  ),
                  Text(
                    movie.description,
                    style: const TextStyle(fontSize: 14),
                  ),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}

