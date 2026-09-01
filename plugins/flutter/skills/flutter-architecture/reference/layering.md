# Layering — template and boundary rules

Adapt to the project's existing structure. If the project already has a working convention, follow it
rather than importing this one.

---

## Directory template

```
lib/
  main.dart
  app.dart                       # MaterialApp.router, theme, locale
  core/
    di/                          # providers / service locator wiring
    error/failure.dart
    network/api_client.dart
    router/app_router.dart
    theme/                       # see flutter-ui-craft
  features/
    invoices/
      domain/
        entities/invoice.dart          # pure Dart. No Flutter, no JSON, no dio.
        repositories/invoice_repository.dart   # abstract interface
        usecases/issue_invoice.dart
      data/
        models/invoice_dto.dart        # fromJson/toJson + toEntity()
        datasources/invoice_remote_ds.dart
        datasources/invoice_local_ds.dart
        repositories/invoice_repository_impl.dart
      presentation/
        screens/invoice_list_screen.dart
        widgets/invoice_tile.dart
        controllers/invoice_list_controller.dart
```

Small features do not need all nine directories. Collapse to `domain/`, `data/`, `presentation/` with
flat files until the count justifies subfolders — structure should follow size, not precede it.

---

## The boundary rules, with the violation each prevents

### 1. `domain` imports nothing outward

```dart
// domain/entities/invoice.dart
class Invoice {
  final String id;
  final int totalCents;
  final InvoiceStatus status;
  const Invoice({required this.id, required this.totalCents, required this.status});
}
```

No `package:flutter/*`, no `dart:io`, no `dio`, no `json_serializable`. If `domain` cannot be tested
with a plain `dart test` and no Flutter binding, it is not a domain layer.

**Violation this prevents:** the entity gains a `fromJson`, the API's field names become your domain
vocabulary, and a backend rename becomes a change in twelve widgets.

### 2. DTOs stay in `data` and convert at the edge

```dart
// data/models/invoice_dto.dart
class InvoiceDto {
  final String id;
  final int total_cents;              // the API's shape, not yours
  final String status;
  InvoiceDto.fromJson(Map<String, dynamic> j) : … ;

  Invoice toEntity() => Invoice(
        id: id,
        totalCents: total_cents,
        status: InvoiceStatus.values.byName(status.toLowerCase()),
      );
}
```

**Violation this prevents:** `invoice.total_cents` appearing in a widget.

### 3. The interface lives in `domain`, the implementation in `data`

```dart
// domain/repositories/invoice_repository.dart
abstract interface class InvoiceRepository {
  Future<Result<List<Invoice>, Failure>> list({required String orgId});
  Future<Result<Invoice, Failure>> issue(String invoiceId);
}
```

```dart
// data/repositories/invoice_repository_impl.dart
class InvoiceRepositoryImpl implements InvoiceRepository {
  InvoiceRepositoryImpl(this._remote, this._local);
  final InvoiceRemoteDataSource _remote;
  final InvoiceLocalDataSource _local;

  @override
  Future<Result<List<Invoice>, Failure>> list({required String orgId}) async {
    try {
      final dtos = await _remote.list(orgId);
      await _local.cache(dtos);                       // owns the caching decision
      return Ok(dtos.map((d) => d.toEntity()).toList());
    } on SocketException {
      final cached = await _local.list(orgId);        // owns the offline fallback
      return cached.isEmpty ? const Err(NetworkFailure()) : Ok(cached.map((d) => d.toEntity()).toList());
    } on DioException catch (e) {
      return Err(ServerFailure(e.response?.statusCode ?? 0));   // converts at the edge
    }
  }
}
```

This repository earns its existence: it maps, caches, falls back offline, and converts exceptions
into typed failures. **A repository doing none of those is a pass-through — delete it.**

### 4. `BuildContext` never leaves `presentation`

```dart
// WRONG — domain now depends on the widget tree
Future<void> issueInvoice(BuildContext context, String id) async { … }

// RIGHT — the use-case takes data; the widget owns the context
final result = await ref.read(issueInvoiceProvider)(id);
if (!context.mounted) return;                  // check after every await
result.when(ok: (_) => context.go('/invoices'), err: (f) => showFailure(context, f));
```

`context.mounted` after an `await` is not optional — the widget may be gone, and using a stale
context throws in release.

### 5. Async state is a union, not three booleans

```dart
sealed class AsyncState<T> { const AsyncState(); }
class Loading<T> extends AsyncState<T> { const Loading(); }
class Data<T>    extends AsyncState<T> { final T value; const Data(this.value); }
class Error_<T>  extends AsyncState<T> { final Failure failure; const Error_(this.failure); }
```

`switch` over a sealed class is exhaustive — the compiler tells you when you forgot the error state,
which is the state everyone forgets. (Riverpod's `AsyncValue` gives you this for free; use it if
Riverpod is the project's choice.)

---

## Dependency injection

Whatever the project uses, the shape is the same: **construct concrete classes at the composition
root and inject interfaces downward.** Nothing below the root reaches for a global.

```dart
// Riverpod
final invoiceRepositoryProvider = Provider<InvoiceRepository>((ref) =>
    InvoiceRepositoryImpl(ref.watch(remoteDsProvider), ref.watch(localDsProvider)));
```

Overriding that one provider in a test replaces the whole data layer with a fake. That is the payoff
for all the indirection above — if the layering does not buy you this, it is not buying anything.

---

## Test seams the layering creates

| Layer | Test with | No need for |
|---|---|---|
| `domain` use-cases | `dart test`, fake repository | Flutter binding, device |
| `data` repository | Mock data sources | Network, real DB |
| `presentation` controller | Fake repository | Widgets |
| Widgets | `testWidgets` + fake controller | Network |

If a test needs a real network call or a running device to cover a business rule, the rule is in the
wrong layer.

---

## Quick violation checks

```bash
# Flutter imports in domain — should return nothing
rg "package:flutter/" lib/features/*/domain/

# BuildContext outside presentation
rg "BuildContext" lib/features/*/domain/ lib/features/*/data/

# JSON keys leaking into widgets
rg "fromJson|toJson" lib/features/*/presentation/

# Widget-returning methods instead of widget classes
rg "Widget _build" lib/
```
