# Country Currency & Exchange API

A RESTful API built with Laravel to fetch, store, and manage country data, including currency codes and exchange rates, with CRUD operations and image summary generation.

## Table of Contents
- [Overview](#overview)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [API Endpoints](#api-endpoints)
- [Error Handling](#error-handling)
- [External APIs](#external-apis)
- [Database Schema](#database-schema)
- [Running the Application](#running-the-application)
- [Testing](#testing)
- [Notes](#notes)

## Overview
This project implements a RESTful API that:
- Fetches country data from the [REST Countries API](https://restcountries.com/v2/all?fields=name,capital,region,population,flag,currencies).
- Retrieves exchange rates from the [Exchange Rates API](https://open.er-api.com/v6/latest/USD).
- Computes an estimated GDP for each country based on population, a random multiplier (1000–2000), and exchange rate.
- Stores and caches data in a MySQL database.
- Provides CRUD operations, filtering, sorting, and a summary image generation feature.
- Handles errors gracefully with consistent JSON responses.

## Features
- **Data Refresh**: Fetch and cache country data and exchange rates via `POST /countries/refresh`.
- **CRUD Operations**:
  - List all countries with filters and sorting (`GET /countries`).
  - Retrieve a single country by name (`GET /countries/:name`).
  - Delete a country record (`DELETE /countries/:name`).
- **Status Check**: View total countries and last refresh timestamp (`GET /status`).
- **Image Generation**: Generate and serve a summary image with total countries, top 5 by estimated GDP, and last refresh timestamp (`GET /countries/image`).
- **Validation**: Enforce required fields (name, population, currency_code where applicable).
- **Error Handling**: Return consistent JSON error responses (400, 404, 500, 503).
- **Currency Handling**: Manage cases with multiple, missing, or unsupported currencies.
- **Database Persistence**: Store data in MySQL with case-insensitive matching for updates.

## Prerequisites
- PHP >= 8.0
- Composer
- MySQL
- Laravel 10.x
- GD PHP extension (for image generation)
- FreeType support in GD (for text rendering in images)
- Arial font file (`arial.ttf`) in the `public/fonts` directory
- Internet access for external API calls

## Installation
1. **Clone the Repository**:
   ```bash
   git clone <repository-url>
   cd country-currency-api
   ```

2. **Install Dependencies**:
   ```bash
   composer install
   ```

3. **Copy Environment File**:
   ```bash
   cp .env.example .env
   ```

4. **Generate Application Key**:
   ```bash
   php artisan key:generate
   ```

5. **Set Up Database**:
   - Create a MySQL database.
   - Update the `.env` file with your database credentials:
     ```env
     DB_CONNECTION=mysql
     DB_HOST=127.0.0.1
     DB_PORT=3306
     DB_DATABASE=your_database_name
     DB_USERNAME=your_username
     DB_PASSWORD=your_password
     ```

6. **Configure API URLs** (optional, defaults are set):
   ```env
   COUNTRIES_API_URL=https://restcountries.com/v2/all?fields=name,capital,region,population,flag,currencies
   EXCHANGE_API_URL=https://open.er-api.com/v6/latest/USD
   ```

7. **Run Migrations**:
   ```bash
   php artisan migrate
   ```

8. **Place Font File**:
   - Ensure the Arial font file (`arial.ttf`) is placed in `public/fonts/`. If not available, download a compatible TTF font and update the path in `CountryService.php` if necessary.

## Configuration
- **Environment Variables**:
  - Configure database settings and API URLs in the `.env` file.
  - Ensure the `storage/app/cache` directory is writable for image generation.
- **Storage Setup**:
  - Run `php artisan storage:link` to create a symbolic link for the storage directory if serving files publicly.

## API Endpoints
| Method | Endpoint                     | Description                                      |
|--------|------------------------------|--------------------------------------------------|
| POST   | `/countries/refresh`         | Fetch and cache country data and exchange rates. |
| GET    | `/countries`                | List all countries with optional filters (`region`, `currency`) and sorting (`gdp_desc`, `gdp_asc`, `name_desc`, `name_asc`, `population_desc`, `population_asc`). |
| GET    | `/countries/{name}`         | Retrieve a country by name (case-insensitive).   |
| DELETE | `/countries/{name}`         | Delete a country record by name.                 |
| GET    | `/status`                   | Show total countries and last refresh timestamp. |
| GET    | `/countries/image`          | Serve the generated summary image.               |

### Example Requests
- **Refresh Countries**:
  ```bash
  curl -X POST http://localhost:8000/api/countries/refresh
  ```
  Response:
  ```json
  {
    "message": "Countries refreshed successfully",
    "last_refreshed_at": "2025-10-26T22:03:00Z"
  }
  ```

- **List Countries (Filtered by Region)**:
  ```bash
  curl "http://localhost:8000/api/countries?region=Africa&sort=gdp_desc"
  ```
  Response:
  ```json
  [
    {
      "id": 1,
      "name": "Nigeria",
      "capital": "Abuja",
      "region": "Africa",
      "population": 206139589,
      "currency_code": "NGN",
      "exchange_rate": 1600.23,
      "estimated_gdp": 25767448125.2,
      "flag_url": "https://flagcdn.com/ng.svg",
      "last_refreshed_at": "2025-10-26T22:03:00Z"
    },
    ...
  ]
  ```

- **Get Status**:
  ```bash
  curl http://localhost:8000/api/status
  ```
  Response:
  ```json
  {
    "total_countries": 250,
    "last_refreshed_at": "2025-10-26T22:03:00Z"
  }
  ```

- **Get Summary Image**:
  ```bash
  curl http://localhost:8000/api/countries/image --output summary.png
  ```

## Error Handling
The API returns consistent JSON error responses:
- **400 Bad Request**: Validation failures (e.g., missing required fields).
  ```json
  {
    "error": "Validation failed",
    "details": {
      "currency_code": "is required"
    }
  }
  ```
- **404 Not Found**: Country or image not found.
  ```json
  {
    "error": "Country not found"
  }
  ```
- **500 Internal Server Error**: Unexpected errors (e.g., image generation failure).
  ```json
  {
    "error": "Internal server error"
  }
  ```
- **503 Service Unavailable**: External API failures or timeouts.
  ```json
  {
    "error": "External data source unavailable",
    "details": "Could not fetch data from Countries API"
  }
  ```

## External APIs
- **Countries API**: `https://restcountries.com/v2/all?fields=name,capital,region,population,flag,currencies`
- **Exchange Rates API**: `https://open.er-api.com/v6/latest/USD`

### Currency Handling
- If a country has multiple currencies, only the first currency code is stored.
- If no currencies are available:
  - `currency_code` and `exchange_rate` are set to `null`.
  - `estimated_gdp` is set to `0`.
- If the currency code is not found in the exchange rates:
  - `exchange_rate` and `estimated_gdp` are set to `null`.
- The country record is still stored in all cases.

## Database Schema
The `countries` table includes:
- `id`: Auto-incrementing primary key.
- `name`: String, required.
- `capital`: String, nullable.
- `region`: String, nullable.
- `population`: Integer, required.
- `currency_code`: String, nullable.
- `exchange_rate`: Float, nullable.
- `estimated_gdp`: Float, nullable (computed as `population * random(1000–2000) / exchange_rate`).
- `flag_url`: String, nullable.
- `last_refreshed_at`: Timestamp, updated on refresh.
- Timestamps: `created_at`, `updated_at`.

Migration example:
```php
Schema::create('countries', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('capital')->nullable();
    $table->string('region')->nullable();
    $table->bigInteger('population');
    $table->string('currency_code')->nullable();
    $table->float('exchange_rate')->nullable();
    $table->float('estimated_gdp')->nullable();
    $table->string('flag_url')->nullable();
    $table->timestamp('last_refreshed_at')->nullable();
    $table->timestamps();
});
```

## Running the Application
1. Start the Laravel development server:
   ```bash
   php artisan serve
   ```
   The API will be available at `http://localhost:8000/api`.

2. Test the refresh endpoint to populate the database:
   ```bash
   curl -X POST http://localhost:8000/api/countries/refresh
   ```

3. Access other endpoints as needed.

## Testing
To test the API:
1. Use tools like Postman or cURL to send requests to the endpoints.
2. Verify the database is populated correctly after a refresh.
3. Check the `storage/app/cache/summary.png` file after a successful refresh.
4. Test error cases (e.g., invalid country name, missing image).

Example test cases:
- Fetch a non-existent country (`GET /countries/InvalidCountry` → 404).
- Call refresh when an external API is down (mock a 503 response).
- Filter countries by region (`GET /countries?region=Africa`).
- Sort countries by population (`GET /countries?sort=population_desc`).

## Notes
- **Performance**: Country data is processed in chunks (50 countries) to optimize database operations.
- **Image Generation**: Requires the GD PHP extension with FreeType support. Ensure the `arial.ttf` font is available in `public/fonts/`.
- **Error Logging**: Image generation failures are logged to Laravel’s log files (`storage/logs`).
- **Caching**: Data is only updated via the `/countries/refresh` endpoint, ensuring controlled cache updates.
- **Case-Insensitive Matching**: Country names are matched case-insensitively for updates and retrieval.
- **Random Multiplier**: A new random value (1000–2000) is generated for each country on every refresh for `estimated_gdp` calculation.
