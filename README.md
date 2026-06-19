!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!Мигр:
php artisan make:migration create_blocks_table
php artisan make:migration create_categories_table

database/migrations/2024_01_01_000001_create_categories_table.php:
~~~
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up()
    {
        Schema::create('categories', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('slug')->unique();
            $table->text('description')->nullable();
            $table->boolean('is_active')->default(true);
            $table->timestamps();
        });
    }

    public function down()
    {
        Schema::dropIfExists('categories');
    }
};
~~~

database/migrations/2024_01_01_000002_create_blocks_table.php:
------------------------------------------------------------------------------------------------------
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up()
    {
        Schema::create('blocks', function (Blueprint $table) {
            $table->id();
            $table->foreignId('category_id')->constrained()->onDelete('cascade');
            $table->string('title');
            $table->string('slug')->unique();
            $table->text('content');
            $table->string('image_url')->nullable();
            $table->integer('sort_order')->default(0);
            $table->boolean('is_published')->default(true);
            $table->json('meta_data')->nullable();
            $table->timestamps();
            
            $table->index(['category_id', 'is_published']);
            $table->index('sort_order');
        });
    }

    public function down()
    {
        Schema::dropIfExists('blocks');
    }
};
------------------------------------------------------------------------------------------------------


Модели:



app/Models/Category.php:
------------------------------------------------------------------------------------------------------
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Category extends Model
{
    use HasFactory;

    protected $fillable = [
        'name',
        'slug',
        'description',
        'is_active'
    ];

    protected $casts = [
        'is_active' => 'boolean'
    ];

    public function blocks()
    {
        return $this->hasMany(Block::class);
    }

    public function publishedBlocks()
    {
        return $this->hasMany(Block::class)->where('is_published', true);
    }
}
------------------------------------------------------------------------------------------------------



app/Models/Block.php:
------------------------------------------------------------------------------------------------------
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Block extends Model
{
    use HasFactory;

    protected $fillable = [
        'category_id',
        'title',
        'slug',
        'content',
        'image_url',
        'sort_order',
        'is_published',
        'meta_data'
    ];

    protected $casts = [
        'is_published' => 'boolean',
        'meta_data' => 'array'
    ];

    public function category()
    {
        return $this->belongsTo(Category::class);
    }

    public function getExcerptAttribute()
    {
        return \Illuminate\Support\Str::words(strip_tags($this->content), 30);
    }

    public function getFormattedContentAttribute()
    {
        return nl2br(e($this->content));
    }
}
------------------------------------------------------------------------------------------------------



 сидеры
------------------------------------------------------------------------------------------------------
php artisan make:seeder DatabaseSeeder
php artisan make:seeder CategorySeeder
php artisan make:seeder BlockSeeder
------------------------------------------------------------------------------------------------------



database/seeders/CategorySeeder.php:
------------------------------------------------------------------------------------------------------
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use App\Models\Category;

class CategorySeeder extends Seeder
{
    public function run()
    {
        $categories = [
            [
                'name' => 'Новости',
                'slug' => 'news',
                'description' => 'Последние новости и обновления'
            ],
            [
                'name' => 'Статьи',
                'slug' => 'articles',
                'description' => 'Полезные статьи и гайды'
            ],
            [
                'name' => 'Образование',
                'slug' => 'education',
                'description' => 'Образовательные материалы'
            ],
            [
                'name' => 'Технологии',
                'slug' => 'technology',
                'description' => 'Технологические новости'
            ]
        ];

        foreach ($categories as $category) {
            Category::create($category);
        }
    }
}
------------------------------------------------------------------------------------------------------



database/seeders/BlockSeeder.php:
------------------------------------------------------------------------------------------------------
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use App\Models\Block;
use App\Models\Category;

class BlockSeeder extends Seeder
{
    public function run()
    {
        $categories = Category::all();

        $blocks = [
            [
                'title' => 'Наш первый блок',
                'slug' => 'first-block',
                'content' => '<h2>Добро пожаловать!</h2><p>Это первый блок информации на нашем сайте.</p>',
                'image_url' => '/images/block1.jpg',
                'sort_order' => 1,
                'is_published' => true
            ],
            [
                'title' => 'Второй информационный блок',
                'slug' => 'second-block',
                'content' => '<h2>Важная информация</h2><p>Здесь содержится важная информация для пользователей.</p>',
                'image_url' => '/images/block2.jpg',
                'sort_order' => 2,
                'is_published' => true
            ],
            [
                'title' => 'Третий блок с контентом',
                'slug' => 'third-block',
                'content' => '<h2>Обновления</h2><p>Новые обновления и функции платформы.</p>',
                'image_url' => '/images/block3.jpg',
                'sort_order' => 3,
                'is_published' => true
            ]
        ];

        foreach ($categories as $category) {
            foreach ($blocks as $index => $blockData) {
                Block::create(array_merge($blockData, [
                    'category_id' => $category->id,
                    'sort_order' => $index + 1,
                    'meta_data' => ['author' => 'Admin', 'views' => rand(0, 100)]
                ]));
            }
        }
    }
}
------------------------------------------------------------------------------------------------------


database/seeders/DatabaseSeeder.php:
------------------------------------------------------------------------------------------------------
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run()
    {
        $this->call([
            CategorySeeder::class,
            BlockSeeder::class,
        ]);
    }
}
------------------------------------------------------------------------------------------------------


контроллер:
php artisan make:controller HomeController
php artisan make:controller BlockController
php artisan make:controller CategoryController

app/Http/Controllers/HomeController.php:
------------------------------------------------------------------------------------------------------
<?php

namespace App\Http\Controllers;

use App\Models\Block;
use App\Models\Category;
use Illuminate\Http\Request;

class HomeController extends Controller
{
    public function index()
    {
        $blocks = Block::with('category')
            ->where('is_published', true)
            ->orderBy('sort_order')
            ->take(9)
            ->get();

        $categories = Category::with(['blocks' => function($query) {
            $query->where('is_published', true)->orderBy('sort_order');
        }])->where('is_active', true)->get();

        $recentBlocks = Block::where('is_published', true)
            ->latest()
            ->take(3)
            ->get();

        return view('home', compact('blocks', 'categories', 'recentBlocks'));
    }
}
------------------------------------------------------------------------------------------------------

app/Http/Controllers/BlockController.php:
------------------------------------------------------------------------------------------------------
<?php

namespace App\Http\Controllers;

use App\Models\Block;
use App\Models\Category;
use Illuminate\Http\Request;

class BlockController extends Controller
{
    public function index()
    {
        $blocks = Block::with('category')
            ->where('is_published', true)
            ->orderBy('sort_order')
            ->paginate(9);

        return view('blocks.index', compact('blocks'));
    }

    public function show($slug)
    {
        $block = Block::with('category')
            ->where('slug', $slug)
            ->where('is_published', true)
            ->firstOrFail();

        $relatedBlocks = Block::where('category_id', $block->category_id)
            ->where('id', '!=', $block->id)
            ->where('is_published', true)
            ->take(3)
            ->get();

        return view('blocks.show', compact('block', 'relatedBlocks'));
    }

    public function byCategory($categorySlug)
    {
        $category = Category::where('slug', $categorySlug)
            ->where('is_active', true)
            ->firstOrFail();

        $blocks = Block::where('category_id', $category->id)
            ->where('is_published', true)
            ->orderBy('sort_order')
            ->paginate(9);

        return view('blocks.by-category', compact('category', 'blocks'));
    }
}
------------------------------------------------------------------------------------------------------

app/Http/Controllers/CategoryController.php:
------------------------------------------------------------------------------------------------------
<?php

namespace App\Http\Controllers;

use App\Models\Category;
use Illuminate\Http\Request;

class CategoryController extends Controller
{
    public function index()
    {
        $categories = Category::withCount(['blocks' => function($query) {
            $query->where('is_published', true);
        }])->where('is_active', true)->get();

        return view('categories.index', compact('categories'));
    }

    public function show($slug)
    {
        $category = Category::where('slug', $slug)
            ->where('is_active', true)
            ->with(['blocks' => function($query) {
                $query->where('is_published', true)->orderBy('sort_order');
            }])
            ->firstOrFail();

        return view('categories.show', compact('category'));
    }
}
------------------------------------------------------------------------------------------------------


routes/web.php:
------------------------------------------------------------------------------------------------------
<?php

use App\Http\Controllers\HomeController;
use App\Http\Controllers\BlockController;
use App\Http\Controllers\CategoryController;
use Illuminate\Support\Facades\Route;

Route::get('/', [HomeController::class, 'index'])->name('home');

Route::prefix('blocks')->name('blocks.')->group(function () {
    Route::get('/', [BlockController::class, 'index'])->name('index');
    Route::get('/category/{categorySlug}', [BlockController::class, 'byCategory'])->name('by-category');
    Route::get('/{slug}', [BlockController::class, 'show'])->name('show');
});

Route::prefix('categories')->name('categories.')->group(function () {
    Route::get('/', [CategoryController::class, 'index'])->name('index');
    Route::get('/{slug}', [CategoryController::class, 'show'])->name('show');
});
------------------------------------------------------------------------------------------------------


resources/views/layouts/app.blade.php:
------------------------------------------------------------------------------------------------------
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>@yield('title', config('app.name'))</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <style>
        .block-card {
            transition: transform 0.3s, box-shadow 0.3s;
            height: 100%;
        }
        .block-card:hover {
            transform: translateY(-5px);
            box-shadow: 0 10px 20px rgba(0,0,0,0.15);
        }
        .block-image {
            height: 200px;
            object-fit: cover;
            background: #f8f9fa;
        }
        .category-badge {
            background: #e9ecef;
            color: #495057;
            padding: 5px 15px;
            border-radius: 20px;
            font-size: 0.85rem;
        }
        .footer {
            background: #2c3e50;
            color: white;
            padding: 30px 0;
            margin-top: 50px;
        }
        .navbar-brand {
            font-weight: bold;
            font-size: 1.5rem;
        }
        .hero-section {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 60px 0;
            margin-bottom: 40px;
            border-radius: 15px;
        }
    </style>
</head>
<body>
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
        <div class="container">
            <a class="navbar-brand" href="{{ route('home') }}">Blocks Project</a>
            <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarNav">
                <span class="navbar-toggler-icon"></span>
            </button>
            <div class="collapse navbar-collapse" id="navbarNav">
                <ul class="navbar-nav me-auto">
                    <li class="nav-item">
                        <a class="nav-link" href="{{ route('home') }}">Главная</a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="{{ route('blocks.index') }}">Все блоки</a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="{{ route('categories.index') }}">Категории</a>
                    </li>
                </ul>
            </div>
        </div>
    </nav>

    <main class="py-4">
        @yield('content')
    </main>

    <footer class="footer">
        <div class="container text-center">
            <p class="mb-0">&copy; {{ date('Y') }} Blocks Project. Все права защищены.</p>
        </div>
    </footer>

    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
------------------------------------------------------------------------------------------------------


resources/views/home.blade.php:
------------------------------------------------------------------------------------------------------
@extends('layouts.app')

@section('title', 'Главная страница')

@section('content')
    <div class="container">
        <!-- Hero Section -->
        <div class="hero-section text-center">
            <h1 class="display-4">Добро пожаловать в Blocks Project</h1>
            <p class="lead">Информационные блоки по категориям</p>
        </div>

        <!-- Categories -->
        @if($categories->isNotEmpty())
        <div class="mb-5">
            <h2 class="mb-4">Категории</h2>
            <div class="row">
                @foreach($categories as $category)
                    <div class="col-md-3 mb-3">
                        <a href="{{ route('categories.show', $category->slug) }}" 
                           class="text-decoration-none">
                            <div class="card block-card text-center">
                                <div class="card-body">
                                    <h5 class="card-title">{{ $category->name }}</h5>
                                    <p class="card-text small">{{ $category->description }}</p>
                                    <span class="category-badge">
                                        {{ $category->blocks->count() }} блоков
                                    </span>
                                </div>
                            </div>
                        </a>
                    </div>
                @endforeach
            </div>
        </div>
        @endif

        <!-- Main Blocks -->
        <h2 class="mb-4">Популярные блоки</h2>
        <div class="row">
            @forelse($blocks as $block)
                <div class="col-md-4 mb-4">
                    <div class="card block-card">
                        @if($block->image_url)
                            <img src="{{ $block->image_url }}" class="card-img-top block-image" alt="{{ $block->title }}">
                        @else
                            <div class="block-image d-flex align-items-center justify-content-center bg-light">
                                <span class="text-muted">Нет изображения</span>
                            </div>
                        @endif
                        <div class="card-body">
                            <span class="category-badge">{{ $block->category->name }}</span>
                            <h5 class="card-title mt-2">{{ $block->title }}</h5>
                            <p class="card-text">{{ $block->excerpt }}</p>
                            <a href="{{ route('blocks.show', $block->slug) }}" class="btn btn-primary">Подробнее</a>
                        </div>
                        <div class="card-footer text-muted">
                            <small>Создан: {{ $block->created_at->format('d.m.Y') }}</small>
                        </div>
                    </div>
                </div>
            @empty
                <div class="col-12">
                    <div class="alert alert-info">Блоки пока не добавлены</div>
                </div>
            @endforelse
        </div>

        <!-- Recent Blocks -->
        @if($recentBlocks->isNotEmpty())
        <div class="mt-5">
            <h2 class="mb-4">Новые блоки</h2>
            <div class="row">
                @foreach($recentBlocks as $block)
                    <div class="col-md-4 mb-4">
                        <div class="card block-card">
                            <div class="card-body">
                                <h5 class="card-title">{{ $block->title }}</h5>
                                <p class="card-text">{{ $block->excerpt }}</p>
                                <a href="{{ route('blocks.show', $block->slug) }}" class="btn btn-outline-primary">Читать</a>
                            </div>
                        </div>
                    </div>
                @endforeach
            </div>
        </div>
        @endif
    </div>
@endsection
------------------------------------------------------------------------------------------------------



Запуск проекта
------------------------------------------------------------------------------------------------------
# Запуск миграций
php artisan migrate

# Запуск сидеров
php artisan db:seed

# Запуск сервера
php artisan serve
------------------------------------------------------------------------------------------------------

resources/views/blocks/show.blade.php:
------------------------------------------------------------------------------------------------------
@extends('layouts.app')

@section('title', $block->title)

@section('content')
<div class="container">
    <nav aria-label="breadcrumb">
        <ol class="breadcrumb">
            <li class="breadcrumb-item"><a href="{{ route('home') }}">Главная</a></li>
            <li class="breadcrumb-item"><a href="{{ route('categories.show', $block->category->slug) }}">{{ $block->category->name }}</a></li>
            <li class="breadcrumb-item active">{{ $block->title }}</li>
        </ol>
    </nav>

    <div class="card">
        @if($block->image_url)
            <img src="{{ $block->image_url }}" class="card-img-top" alt="{{ $block->title }}">
        @endif
        <div class="card-body">
            <h1 class="card-title">{{ $block->title }}</h1>
            <div class="mb-3">
                <span class="category-badge">{{ $block->category->name }}</span>
                <small class="text-muted ms-2">Опубликован: {{ $block->created_at->format('d.m.Y H:i') }}</small>
            </div>
            <div class="card-text">
                {!! $block->formatted_content !!}
            </div>
        </div>
    </div>

    @if($relatedBlocks->isNotEmpty())
    <div class="mt-5">
        <h3>Похожие блоки</h3>
        <div class="row">
            @foreach($relatedBlocks as $related)
                <div class="col-md-4">
                    <div class="card block-card">
                        <div class="card-body">
                            <h5 class="card-title">{{ $related->title }}</h5>
                            <a href="{{ route('blocks.show', $related->slug) }}" class="btn btn-sm btn-primary">Читать</a>
                        </div>
                    </div>
                </div>
            @endforeach
        </div>
    </div>
    @endif
</div>
@endsection
------------------------------------------------------------------------------------------------------




------------------------------------------------------------------------------------------------------

------------------------------------------------------------------------------------------------------

