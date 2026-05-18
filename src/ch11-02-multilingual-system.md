# Multilingual System Design

## Why i18n Matters

**i18n** stands for **internationalization** (18 letters between i and n). It means designing your application to support multiple languages. If you are building a global application, you cannot assume everyone speaks English.

I used to think i18n was only for big companies. Then I built an ecommerce API for a client who needed it in English, Spanish, and Arabic. Suddenly I had to think about language in every response, every error message, and every notification.

The good news: adding i18n to a Go API is straightforward if you design for it from the start. The bad news: adding it later is painful.

## The Basic Approach

The idea is simple: instead of hardcoding strings in your code, you store them in a **translation map** and look them up by key. Each language has its own set of translations.

```go
// Instead of this:
message := "Product not found"

// Do this:
message := t.Get("en", "errors.product_not_found")
```

## Translation Map

The simplest approach is a nested map: `map[language][key]translation`:

```go
package i18n

import "fmt"

type Translator struct {
    translations map[string]map[string]string
    defaultLang  string
}

func NewTranslator(defaultLang string) *Translator {
    return &Translator{
        translations: make(map[string]map[string]string),
        defaultLang:  defaultLang,
    }
}

// Add translations for a language
func (t *Translator) AddTranslations(lang string, translations map[string]string) {
    if t.translations[lang] == nil {
        t.translations[lang] = make(map[string]string)
    }
    for key, value := range translations {
        t.translations[lang][key] = value
    }
}

// Get a translation by language and key
func (t *Translator) Get(lang, key string) string {
    // Try the requested language
    if translations, ok := t.translations[lang]; ok {
        if value, ok := translations[key]; ok {
            return value
        }
    }

    // Fall back to default language
    if translations, ok := t.translations[t.defaultLang]; ok {
        if value, ok := translations[key]; ok {
            return value
        }
    }

    // Return the key itself if no translation found
    return key
}

// Get with interpolation
func (t *Translator) Getf(lang, key string, args ...interface{}) string {
    template := t.Get(lang, key)
    return fmt.Sprintf(template, args...)
}
```

## Loading Translations

You can store translations in JSON files:

```json
// locales/en.json
{
    "errors.product_not_found": "Product not found",
    "errors.unauthorized": "You are not authorized to perform this action",
    "errors.validation_failed": "Validation failed",
    "success.order_created": "Order created successfully",
    "success.product_updated": "Product updated successfully",
    "welcome": "Welcome to our store",
    "greeting": "Hello, %s!"
}
```

```json
// locales/es.json
{
    "errors.product_not_found": "Producto no encontrado",
    "errors.unauthorized": "No estas autorizado para realizar esta accion",
    "errors.validation_failed": "Validacion fallida",
    "success.order_created": "Pedido creado exitosamente",
    "success.product_updated": "Producto actualizado exitosamente",
    "welcome": "Bienvenido a nuestra tienda",
    "greeting": "Hola, %s!"
}
```

```json
// locales/fr.json
{
    "errors.product_not_found": "Produit non trouve",
    "errors.unauthorized": "Vous n etes pas autorise a effectuer cette action",
    "errors.validation_failed": "Validation echouee",
    "success.order_created": "Commande creee avec succes",
    "success.product_updated": "Produit mis a jour avec succes",
    "welcome": "Bienvenue dans notre boutique",
    "greeting": "Bonjour, %s!"
}
```

Load them in your application:

```go
package i18n

import (
    "encoding/json"
    "os"
)

func LoadFromDirectory(dir string, defaultLang string) (*Translator, error) {
    t := NewTranslator(defaultLang)

    files, err := os.ReadDir(dir)
    if err != nil {
        return nil, fmt.Errorf("failed to read locales directory: %w", err)
    }

    for _, file := range files {
        if file.IsDir() {
            continue
        }

        // Extract language from filename (e.g., "en.json" -> "en")
        lang := strings.TrimSuffix(file.Name(), ".json")
        if lang == "" {
            continue
        }

        data, err := os.ReadFile(filepath.Join(dir, file.Name()))
        if err != nil {
            return nil, fmt.Errorf("failed to read %s: %w", file.Name(), err)
        }

        var translations map[string]string
        if err := json.Unmarshal(data, &translations); err != nil {
            return nil, fmt.Errorf("failed to parse %s: %w", file.Name(), err)
        }

        t.AddTranslations(lang, translations)
    }

    return t, nil
}
```

## Detecting Language from Requests

There are two common ways to detect the user's preferred language:

**1. Accept-Language Header** - The browser sends this header automatically:

```go
func GetLanguageFromHeader(r *http.Request, supportedLangs []string, defaultLang string) string {
    acceptLang := r.Header.Get("Accept-Language")
    if acceptLang == "" {
        return defaultLang
    }

    // Parse Accept-Language header (simplified)
    // Real header looks like: "en-US,en;q=0.9,es;q=0.8"
    for _, part := range strings.Split(acceptLang, ",") {
        lang := strings.TrimSpace(strings.Split(part, ";")[0])
        lang = strings.Split(lang, "-")[0] // "en-US" -> "en"
        for _, supported := range supportedLangs {
            if lang == supported {
                return lang
            }
        }
    }

    return defaultLang
}
```

**2. URL Parameter** - The client explicitly specifies the language:

```go
func GetLanguageFromQuery(r *http.Request, supportedLangs []string, defaultLang string) string {
    lang := r.URL.Query().Get("lang")
    if lang == "" {
        return defaultLang
    }

    for _, supported := range supportedLangs {
        if lang == supported {
            return lang
        }
    }

    return defaultLang
}
```

## Middleware for Language Detection

Create middleware that detects the language and adds it to the request context:

```go
package middleware

import (
    "context"
    "net/http"
)

type contextKey string

const LanguageKey contextKey = "language"

func LanguageDetection(supportedLangs []string, defaultLang string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Check URL parameter first
            lang := r.URL.Query().Get("lang")

            // Fall back to Accept-Language header
            if lang == "" {
                lang = GetLanguageFromHeader(r, supportedLangs, defaultLang)
            }

            // Validate the language
            valid := false
            for _, supported := range supportedLangs {
                if lang == supported {
                    valid = true
                    break
                }
            }
            if !valid {
                lang = defaultLang
            }

            // Add to context
            ctx := context.WithValue(r.Context(), LanguageKey, lang)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

## Using Translations in Responses

Now your handlers can return localized messages:

```go
func (h *ProductHandler) GetProduct(w http.ResponseWriter, r *http.Request) {
    lang := r.Context().Value(LanguageKey).(string)

    productID, _ := strconv.ParseInt(r.PathValue("id"), 10, 64)
    product, err := h.service.GetProduct(productID)
    if err != nil {
        respondJSON(w, http.StatusNotFound, map[string]string{
            "error":   h.translator.Get(lang, "errors.product_not_found"),
            "code":    "PRODUCT_NOT_FOUND",
            "message": h.translator.Getf(lang, "errors.product_not_found_id", productID),
        })
        return
    }

    respondJSON(w, http.StatusOK, map[string]interface{}{
        "data":    product,
        "message": h.translator.Get(lang, "success.product_retrieved"),
    })
}
```

## JSON Response with Localized Messages

Here is a standard response format that includes localization:

```go
type APIResponse struct {
    Data    interface{} `json:"data,omitempty"`
    Error   string      `json:"error,omitempty"`
    Code    string      `json:"code,omitempty"`
    Message string      `json:"message"`
    Lang    string      `json:"lang"`
}

func respondLocalized(w http.ResponseWriter, status int, lang string, data interface{}, messageKey string, args ...interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)

    response := APIResponse{
        Data:    data,
        Message: translator.Getf(lang, messageKey, args...),
        Lang:    lang,
    }

    json.NewEncoder(w).Encode(response)
}
```

## Storing Translations in the Database

For larger applications, you might store translations in the database instead of JSON files:

```sql
CREATE TABLE translations (
    id SERIAL PRIMARY KEY,
    lang VARCHAR(5) NOT NULL,
    key VARCHAR(255) NOT NULL,
    value TEXT NOT NULL,
    UNIQUE(lang, key)
);
```

This lets you update translations without redeploying the application. But it adds a database query for every translation. Cache the translations in memory to avoid this.

## My Experience

I added i18n to a project after it was already built. It was painful. Every hardcoded string had to be found and replaced. Error messages were scattered across 50 files. Some were in handlers, some in services, some in middleware.

The second time, I designed for i18n from the start. Every user-facing string went through the translator. It was more work initially, but when the client asked for French support, I just added a JSON file. No code changes needed.

The lesson: **always design for i18n from the beginning**, even if you only support one language right now. It is much harder to add later.
