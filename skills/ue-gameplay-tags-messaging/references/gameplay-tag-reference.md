# Gameplay Tag API Reference

Full surface of the `GameplayTags` runtime module. Target engine: UE 5.8.
Headers live in `Engine/Source/Runtime/GameplayTags/Classes` and `.../Public`.

Build.cs: `PublicDependencyModuleNames.Add("GameplayTags");` — the module itself depends on
`Core`, `CoreUObject`, `Engine` and `DeveloperSettings`.

| Header | Holds |
|---|---|
| `GameplayTagContainer.h` | `FGameplayTag`, `FGameplayTagContainer`, `FGameplayTagQuery`, `FGameplayTagQueryExpression` |
| `GameplayTagsManager.h` | `UGameplayTagsManager`, `FGameplayTagTableRow`, `FGameplayTagSource`, `FGameplayTagNode` |
| `NativeGameplayTags.h` | The `UE_*_GAMEPLAY_TAG*` macros and `FNativeGameplayTag` |
| `GameplayTagsSettings.h` | `UGameplayTagsSettings`, `UGameplayTagsList`, `UGameplayTagsDeveloperSettings` |
| `GameplayTagAssetInterface.h` | `IGameplayTagAssetInterface` |
| `BlueprintGameplayTagLibrary.h` | `UBlueprintGameplayTagLibrary` |
| `GameplayTagRedirectors.h` | `FGameplayTagRedirect`, `FGameplayTagRedirectors` |
| `GameplayTagsModule.h` | `IGameplayTagsModule` (two change delegates only) |

---

## FGameplayTag

A `USTRUCT(BlueprintType)` wrapping a single `FName`. Copy by value.

```cpp
static FGameplayTag RequestGameplayTag(const FName& TagName, bool ErrorIfNotFound = true);
static bool IsValidGameplayTagString(const FString& TagString, FText* OutError = nullptr, FString* OutFixedString = nullptr);

bool MatchesTag(const FGameplayTag& TagToCheck) const;
bool MatchesTagExact(const FGameplayTag& TagToCheck) const;
int32 MatchesTagDepth(const FGameplayTag& TagToCheck) const;
bool MatchesAny(const FGameplayTagContainer& ContainerToCheck) const;
bool MatchesAnyExact(const FGameplayTagContainer& ContainerToCheck) const;

bool IsValid() const;
FGameplayTagContainer GetSingleTagContainer() const;
FGameplayTag RequestDirectParent() const;
FGameplayTagContainer GetGameplayTagParents() const;
void ParseParentTags(TArray<FGameplayTag>& UniqueParentTags) const;
FName GetTagLeafName() const;
FString ToString() const;
FName GetTagName() const;

bool NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess);
void PostSerialize(const FArchive& Ar);
void FromExportString(const FString& ExportString, int32 PortFlags = 0);

static const FGameplayTag EmptyTag;
```

`RequestGameplayTag` ensures when `ErrorIfNotFound` is true and the tag is unregistered; pass `false`
for optional lookups (reading user data, tolerating a disabled plugin) and check `IsValid()`.

`MatchesTagDepth` returns how many hierarchy levels two tags share — useful for scoring a
"best match" among candidate handlers.

`ParseParentTags` parses the name without consulting the manager, so it also works for tags whose
source has not loaded yet. `GetGameplayTagParents` goes through the manager and returns a container.

`FGameplayTag`'s constructor from `FName` is protected: only `UGameplayTagsManager`,
`FGameplayTagRedirectors`, `FNativeGameplayTag` and the container/node types can build one from raw
text. Everything else goes through `RequestGameplayTag` or a native tag variable.

## FGameplayTagContainer

Holds an explicit `GameplayTags` array plus a transient `ParentTags` array, so parent-aware checks are
array searches. `Num()` counts the explicit array only.

```cpp
explicit FGameplayTagContainer(const FGameplayTag& Tag);
template<class AllocatorType>
static FGameplayTagContainer CreateFromArray(const TArray<FGameplayTag, AllocatorType>& SourceTags);

bool HasTag(const FGameplayTag& TagToCheck) const;
bool HasTagExact(const FGameplayTag& TagToCheck) const;
bool HasAny(const FGameplayTagContainer& ContainerToCheck) const;
bool HasAnyExact(const FGameplayTagContainer& ContainerToCheck) const;
bool HasAll(const FGameplayTagContainer& ContainerToCheck) const;
bool HasAllExact(const FGameplayTagContainer& ContainerToCheck) const;
bool MatchesQuery(const struct FGameplayTagQuery& Query) const;

FGameplayTagContainer Filter(const FGameplayTagContainer& OtherContainer) const;
FGameplayTagContainer FilterExact(const FGameplayTagContainer& OtherContainer) const;
FGameplayTagContainer GetGameplayTagParents() const;

void AddTag(const FGameplayTag& TagToAdd);
void AddTagFast(const FGameplayTag& TagToAdd);
bool AddLeafTag(const FGameplayTag& TagToAdd);
bool RemoveTag(const FGameplayTag& TagToRemove, bool bDeferParentTags = false);
void RemoveTags(const FGameplayTagContainer& TagsToRemove);
void AppendTags(FGameplayTagContainer const& Other);
void AppendMatchingTags(FGameplayTagContainer const& OtherA, FGameplayTagContainer const& OtherB);
void Reset(int32 Slack = 0);
void FillParentTags();

int32 Num() const;
bool IsValid() const;
bool IsEmpty() const;
bool IsValidIndex(int32 Index) const;
FGameplayTag GetByIndex(int32 Index) const;
FGameplayTag First() const;
FGameplayTag Last() const;
const TArray<FGameplayTag>& GetGameplayTagArray() const;
TArray<FGameplayTag>::TConstIterator CreateConstIterator() const;

FString ToString() const;
FString ToStringSimple(bool bQuoted = false) const;
TArray<FString> ToStringsMaxLen(int32 MaxLen) const;
FText ToMatchingText(EGameplayContainerMatchType MatchType, bool bInvertCondition) const;
bool NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess);

static const FGameplayTagContainer EmptyContainer;
```

Notes that bite:

- **The constructor from a single tag is `explicit`**, on purpose: `SomeFunctionTakingContainer(MyTag)`
  will not compile. Use `MyTag.GetSingleTagContainer()` or `FGameplayTagContainer(MyTag)`.
- `Filter` keeps the tags of **this** container whose own parent chain matches an explicit tag in
  `OtherContainer`, so `{A.1, C}.Filter({A})` returns `{A.1}` — the child name, not `A`. `FilterExact`
  compares names and is the true intersection.
- `AppendMatchingTags(OtherA, OtherB)` adds the tags of `OtherA` whose parent chain matches an explicit
  tag in `OtherB`, so the result carries `OtherA`'s names. For a strict intersection use `FilterExact`;
  for a disjunctive union, `AppendTags` to build the union, `FilterExact` to build the intersection,
  then `RemoveTags` the intersection from the union.
- `RemoveTag(Tag, bDeferParentTags = true)` skips the `FillParentTags` rebuild — call `FillParentTags()`
  yourself after a batch of removals.
- The container supports range-based `for`, which iterates the explicit tags only.
- `ToStringSimple(true)` quotes each tag, which is what you want when building a log line that will be
  parsed again.

`EGameplayContainerMatchType` is `Any` or `All`, and is used only by `ToMatchingText` for building
human-readable descriptions.

## FGameplayTagQuery

```cpp
bool Matches(FGameplayTagContainer const& Tags) const;
bool IsEmpty() const;
void Clear();
void Build(struct FGameplayTagQueryExpression& RootQueryExpr, FString InUserDescription = FString());
static FGameplayTagQuery BuildQuery(struct FGameplayTagQueryExpression& RootQueryExpr, FString InDescription = FString());
void GetQueryExpr(struct FGameplayTagQueryExpression& OutExpr) const;
void ReplaceTagsFast(FGameplayTagContainer const& Tags);
void ReplaceTagFast(FGameplayTag const& Tag);
void SetUserDescription(const FString& InUserDescription);
const FString& GetDescription() const;
const TArray<FGameplayTag>& GetGameplayTagArray() const;

static FGameplayTagQuery MakeQuery_MatchAnyTags(FGameplayTagContainer const& InTags);
static FGameplayTagQuery MakeQuery_MatchAllTags(FGameplayTagContainer const& InTags);
static FGameplayTagQuery MakeQuery_MatchNoTags(FGameplayTagContainer const& InTags);
static FGameplayTagQuery MakeQuery_ExactMatchAnyTags(FGameplayTagContainer const& InTags);
static FGameplayTagQuery MakeQuery_ExactMatchAllTags(FGameplayTagContainer const& InTags);
static FGameplayTagQuery MakeQuery_MatchTag(FGameplayTag const& InTag);

static const FGameplayTagQuery EmptyQuery;
```

The query stores a `TagDictionary` plus a compiled `QueryTokenStream`. `Build`/`BuildQuery` emit that
stream; `Matches` walks it. `EGameplayTagQueryExprType` enumerates the node kinds: `AnyTagsMatch`,
`AllTagsMatch`, `NoTagsMatch`, `AnyExprMatch`, `AllExprMatch`, `NoExprMatch`, `AnyTagsExactMatch`,
`AllTagsExactMatch`.

`FGameplayTagQueryExpression` is a plain struct (not a `USTRUCT`) with fluent setters returning
`FGameplayTagQueryExpression&`, which is why temporaries chain:

```cpp
FGameplayTagQuery Q = FGameplayTagQuery::BuildQuery(
    FGameplayTagQueryExpression()
        .AnyExprMatch()
        .AddExpr(FGameplayTagQueryExpression().AllTagsMatch().AddTag(TAG_MyGame_Damage_Fire))
        .AddExpr(FGameplayTagQueryExpression().AllTagsExactMatch().AddTag(TAG_MyGame_State_Stunned)),
    TEXT("Fire or exactly Stunned"));
```

`AddTag` has three overloads — `FGameplayTag`, `FName`, `const TCHAR*` — and `AddTags` takes a
container. `AddExpr` requires an expression-set type (`AnyExprMatch`, `AllExprMatch`, `NoExprMatch`);
mixing tag and expression sets on one node triggers the `ensure` in `UsesTagSet()` / `UsesExprSet()`.

`ConvertToJsonObject` / `MakeFromJsonObject` round-trip an expression through JSON, which is how
tooling stores queries outside a `UPROPERTY`.

In the editor a `FGameplayTagQuery` property is edited through `UEditableGameplayTagQuery` and the
`UEditableGameplayTagQueryExpression` subclasses, one per expression type (`AnyTagsMatch`,
`AllTagsMatch`, `NoTagsMatch`, `AnyTagsExactMatch`, `AllTagsExactMatch`, `AnyExprMatch`,
`AllExprMatch`, `NoExprMatch`). These are
`Transient`, editor-only and converted back to the token stream on save — never reference them from
runtime code.

## UGameplayTagsManager

`UCLASS(config=Engine)` singleton. `UGameplayTagsManager::Get()` constructs it on first use;
`GetIfAllocated()` returns null instead of constructing, for shutdown paths.

```cpp
static UGameplayTagsManager& Get();
static UGameplayTagsManager* GetIfAllocated();

FGameplayTag RequestGameplayTag(FName TagName, bool ErrorIfNotFound = true) const;
void RequestGameplayTagContainer(const TArray<FString>& TagStrings, FGameplayTagContainer& OutTagsContainer, bool bErrorIfNotFound = true) const;
FGameplayTag FindGameplayTagFromPartialString_Slow(FString PartialString) const;
bool IsValidGameplayTagString(const FString& TagString, FText* OutError = nullptr, FString* OutFixedString = nullptr);

FGameplayTag AddNativeGameplayTag(FName TagName, const FString& TagDevComment = TEXT("(Native)"));
void DoneAddingNativeTags();
static FDelegateHandle CallOrRegister_OnAddNativeTagsDelegate(const FSimpleMulticastDelegate::FDelegate& Delegate);
static FDelegateHandle CallOrRegister_OnDoneAddingNativeTagsDelegate(const FSimpleMulticastDelegate::FDelegate& Delegate);
static void UnregisterNativeTagDelegate(FDelegateHandle DelegateHandle);

FGameplayTagContainer RequestGameplayTagParents(const FGameplayTag& GameplayTag) const;
bool ExtractParentTags(const FGameplayTag& GameplayTag, TArray<FGameplayTag>& UniqueParentTags) const;
FGameplayTagContainer RequestGameplayTagChildren(const FGameplayTag& GameplayTag) const;
FGameplayTag RequestGameplayTagDirectParent(const FGameplayTag& GameplayTag) const;
void RequestAllGameplayTags(FGameplayTagContainer& TagContainer, bool OnlyIncludeDictionaryTags) const;

TSharedPtr<FGameplayTagNode> FindTagNode(const FGameplayTag& GameplayTag) const;
TSharedPtr<FGameplayTagNode> FindTagNode(FName TagName) const;
void SplitGameplayTagFName(const FGameplayTag& Tag, TArray<FName>& OutNames) const;
int32 GameplayTagsMatchDepth(const FGameplayTag& GameplayTagOne, const FGameplayTag& GameplayTagTwo) const;
int32 GetNumberOfTagNodes(const FGameplayTag& GameplayTag) const;

void LoadGameplayTagTables(bool bAllowAsyncLoad = false);
void AddTagIniSearchPath(const FString& RootDir, const TSet<FString>* PluginConfigsCache = nullptr);
bool RemoveTagIniSearchPath(const FString& RootDir);
void GetTagSourceSearchPaths(TArray<FString>& OutPaths);
int32 GetNumTagSourceSearchPaths();
const FGameplayTagSource* FindTagSource(FName TagSourceName) const;
void FindTagSourcesWithType(EGameplayTagSourceType TagSourceType, TArray<const FGameplayTagSource*>& OutArray) const;
void FindTagsWithSource(FStringView PackageNameOrPath, TArray<FGameplayTag>& OutTags) const;
void GetAllTagsFromSource(FName TagSource, TArray<TSharedPtr<FGameplayTagNode>>& OutTagArray) const;

bool ShouldImportTagsFromINI() const;
bool ShouldWarnOnInvalidTags() const;
bool ShouldUseFastReplication() const;
bool ShouldUseDynamicReplication() const;
bool ShouldUnloadTags() const;
void SetShouldUnloadTagsOverride(bool bShouldUnloadTags);
void ClearShouldUnloadTagsOverride();

FName GetTagNameFromNetIndex(FGameplayTagNetIndex Index) const;
FGameplayTagNetIndex GetNetIndexFromTag(const FGameplayTag& InTag) const;
int32 GetNetIndexTrueBitNum() const;
int32 GetNetIndexFirstBitSegment() const;
FGameplayTagNetIndex GetInvalidTagNetIndex() const;
uint32 GetNetworkGameplayTagNodeIndexHash() const;
```

`FGameplayTagNetIndex` is `uint16`; `INVALID_TAGNETINDEX` is `MAX_uint16`.

`RequestAllGameplayTags(Container, /*OnlyIncludeDictionaryTags*/ true)` skips tags that exist only
because a child implied them — the usual choice for building an editor list.

`FindTagNode` returns a `TSharedPtr<FGameplayTagNode>` and takes the manager's lock; in the editor it
also follows redirectors. `FGameplayTagNode` exposes `GetSingleTagContainer()`, `GetCompleteTag()`,
`GetCompleteTagName()`, `GetCompleteTagString()`, `GetSimpleTagName()`, `GetChildTagNodes()`,
`GetParentTagNode()`, `GetNetIndex()`, `IsExplicitTag()`, `IsRestrictedGameplayTag()`,
`GetAllowNonRestrictedChildren()`, and (editor-only) `GetDevComment()`, `GetFirstSourceName()`,
`GetAllSourceNames()`.

`FGameplayTagNativeAdder` is an older alternative to the macros: subclass it, override
`virtual void AddTags()` and call `AddNativeGameplayTag` inside. The macros are preferred — they
unregister automatically on module unload.

`UGameplayTagsManager::OnGameplayTagLoadedDelegate` (a `DECLARE_TS_MULTICAST_DELEGATE_OneParam`) fires
for each tag resolved during serialization; `OnFilterGameplayTag` and `OnFilterGameplayTagChildren`
let editor tooling hide tags from pickers.

## Ini and data table sources in detail

`UGameplayTagsList` (`config = GameplayTagsList`) is the base for every ini tag file and owns
`ConfigFileName`, `GameplayTagRedirects` and `GameplayTagList`. `UGameplayTagsSettings`
(`config=GameplayTags, defaultconfig`) derives from it and adds the project-wide settings, so
everything lands under `[/Script/GameplayTags.GameplayTagsSettings]` in `Config/DefaultGameplayTags.ini`.

```ini
[/Script/GameplayTags.GameplayTagsSettings]
ImportTagsFromConfig=True
WarnOnInvalidTags=True
AllowEditorTagUnloading=True
AllowGameTagUnloading=False
FastReplication=False
bDynamicReplication=True
NumBitsForContainerSize=6
NetIndexFirstBitSegment=16
InvalidTagCharacters="\"',"
+CommonlyReplicatedTags=MyGame.State.Stunned
+GameplayTagList=(Tag="MyGame.State.Stunned",DevComment="Character cannot act")
+CategoryRemapping=(BaseCategory="Mover",RemapCategories=("MyGame.Movement"))
+RestrictedConfigFiles=(RestrictedConfigName="DesignTags.ini",Owners=("design-lead"))
```

Extra sources:

- `Config/Tags/*.ini` is added automatically (`FPaths::ProjectConfigDir() / TEXT("Tags")`).
- `AddTagIniSearchPath(RootDir)` registers any other directory; Game Feature plugins use this so their
  tags appear only while active. `RemoveTagIniSearchPath` unregisters, and requires
  `AllowEditorTagUnloading` / `AllowGameTagUnloading` to actually drop the tags.
- `GameplayTagTableList` holds `FSoftObjectPath`s to `UDataTable` assets whose row struct is
  `FGameplayTagTableRow` (`Tag`, `DevComment`). `LoadGameplayTagTables(bAllowAsyncLoad)` loads them.
- Restricted tags use `FRestrictedGameplayTagTableRow` (adds `bAllowNonRestrictedChildren`) stored in a
  `URestrictedGameplayTagsList` file named by `FRestrictedConfigInfo::RestrictedConfigName`, under
  `Config/Tags/`. Only the listed `Owners` are meant to edit them; use this for the top two levels of a
  large hierarchy.

`FGameplayTagSource` describes where a tag came from: `SourceName`, `SourceType`, and the
`UGameplayTagsList` / `URestrictedGameplayTagsList` object backing it. `GetConfigFileName()` returns
the ini path.

### Redirects

```ini
+GameplayTagRedirects=(OldTagName="MyGame.State.Stun",NewTagName="MyGame.State.Stunned")
```

`FGameplayTagRedirect` is `{ FName OldTagName; FName NewTagName; }`. `FGameplayTagRedirectors::Get()`
exposes `RedirectTag(const FName& InTagName, FGameplayTag& OutTag)`, `RefreshTagRedirects()` and
`AddRedirectsFromSource(const FGameplayTagSource*)`, but the class has no `GAMEPLAYTAGS_API`
(`GameplayTagRedirectors.h:45`), so calling it from another module is a link error. Serialization calls
`UGameplayTagsManager::RedirectSingleGameplayTag` / `RedirectTagsForContainer` /
`ImportSingleGameplayTag`, so redirects apply on load, on text import and in the asset registry.

Multi-hop redirects inside one list are flattened when the map is built (with a ten-iteration
recursion guard), so `A → B → C` resolves `A` straight to `C`. A redirect whose `NewTagName` is not a
registered tag still produces an invalid tag, so land every chain on a real name.

## IGameplayTagAssetInterface

```cpp
virtual void GetOwnedGameplayTags(FGameplayTagContainer& TagContainer) const = 0;
virtual bool HasMatchingGameplayTag(FGameplayTag TagToCheck) const;
virtual bool HasAllMatchingGameplayTags(const FGameplayTagContainer& TagContainer) const;
virtual bool HasAnyMatchingGameplayTags(const FGameplayTagContainer& TagContainer) const;
```

Only `GetOwnedGameplayTags` is pure virtual; the three `Has*` functions have engine implementations
that call it, so override them only to add a faster path. The `UINTERFACE` is
`meta=(CannotImplementInterfaceInBlueprint)` — C++ only.

## UBlueprintGameplayTagLibrary

Every static is `BlueprintPure` unless noted, and most are `BlueprintThreadSafe`. The Blueprint side
takes a `bExactMatch` bool where C++ has two differently named functions.

| Function | C++ equivalent |
|---|---|
| `MatchesTag(TagOne, TagTwo, bExactMatch)` | `MatchesTag` / `MatchesTagExact` |
| `MatchesAnyTags(TagOne, OtherContainer, bExactMatch)` | `MatchesAny` / `MatchesAnyExact` |
| `HasTag(TagContainer, Tag, bExactMatch)` | `HasTag` / `HasTagExact` |
| `HasAnyTags(TagContainer, OtherContainer, bExactMatch)` | `HasAny` / `HasAnyExact` |
| `HasAllTags(TagContainer, OtherContainer, bExactMatch)` | `HasAll` / `HasAllExact` |
| `Filter(TagContainer, OtherContainer, bExactMatch)` | `Filter` / `FilterExact` |
| `DoesContainerMatchTagQuery(TagContainer, TagQuery)` | `FGameplayTagContainer::MatchesQuery` |
| `MakeGameplayTagQuery_MatchAnyTags` / `_MatchAllTags` / `_MatchNoTags` | the `MakeQuery_*` statics |
| `AddGameplayTag` / `RemoveGameplayTag` / `AppendGameplayTagContainers` (`BlueprintCallable`) | `AddTag` / `RemoveTag` / `AppendTags` |
| `GetOwnedGameplayTags(TagContainerInterface)` | `IGameplayTagAssetInterface::GetOwnedGameplayTags` |
| `GetAllActorsOfClassMatchingTagQuery` (`BlueprintCallable`) | no direct C++ equivalent |
| `GetDebugStringFromGameplayTag` / `GetDebugStringFromGameplayTagContainer` | `ToString` / `ToStringSimple` |

`IsGameplayTagValid`, `GetTagName`, `MakeLiteralGameplayTag`, `GetNumGameplayTagsInContainer`,
`MakeGameplayTagContainerFromArray`, `MakeGameplayTagContainerFromTag`, `BreakGameplayTagContainer`
and `IsTagQueryEmpty` round out the library.

## Editor filtering metadata

The metadata keys are declared in `namespace GameplayTagsManager` in
`CoreUObject/Public/UObject/ObjectMacros.h`:

| Key | Applies to | Effect |
|---|---|---|
| `Categories` | `FGameplayTag` / `FGameplayTagContainer` properties and function parameters | Restricts the picker to that subtree |
| `GameplayTagFilter` | Function parameters that become Blueprint pins, via `UPARAM(meta = (…))` | Same restriction on the pin's picker |

```cpp
UPROPERTY(EditDefaultsOnly, Category = "Damage", meta = (Categories = "MyGame.Damage"))
FGameplayTag DamageType;

UPROPERTY(EditDefaultsOnly, Category = "Gating", meta = (Categories = "MyGame.State"))
FGameplayTagContainer BlockedByTags;
```

Comma-separate several roots: `meta = (Categories = "MyGame.Damage,MyGame.Element")`.
`UGameplayTagsManager::GetCategoriesMetaFromField` reads `Categories` first and falls back to
`GameplayTagFilter`; `GetCategoriesMetaFromFunction(const UFunction* Func, FName ParamName)` resolves
the filter for a Blueprint pin, and `GetCategoriesMetaFromPropertyHandle` for a details row.
`GetFilteredGameplayRootTags(InFilterString, OutTagArray)` applies such a string to the tag tree.

`FGameplayTagCategoryRemap` in `UGameplayTagsSettings` maps an engine-declared `BaseCategory` onto one
or more project `RemapCategories`, so an engine property whose filter says `Mover` can show your
`MyGame.Movement` tags instead. `UGameplayTagsDeveloperSettings`
(`config=EditorPerProjectUserSettings`, display name "Gameplay Tag Editing") holds the per-user
`DeveloperConfigName` and `FavoriteTagSource` that decide which ini a newly created tag lands in.

`IGameplayTagsModule::OnGameplayTagTreeChanged` and `IGameplayTagsModule::OnTagSettingsChanged` are
`FSimpleMulticastDelegate`s for refreshing editor UI. `UGameplayTagsManager::OnEditorRefreshGameplayTagTree`
is the editor-side rebuild hook. `PushDeferOnGameplayTagTreeChangedBroadcast()` /
`PopDeferOnGameplayTagTreeChangedBroadcast()` batch a burst of registrations into one notification, and
`SetShouldDeferGameplayTagTreeRebuilds(true)` / `ClearShouldDeferGameplayTagTreeRebuilds(bRebuildTree)`
suppress the rebuild itself.

`FGameplayTagCreationWidgetHelper` is an empty `USTRUCT` you embed in another struct to get an inline
"create new tag" widget in the details panel.

## Replication settings reference

| Setting | Default behaviour | When to change |
|---|---|---|
| `FastReplication` | Off. Tags replicate by name | Turn on only when client and server dictionaries are provably identical — no per-side plugins, no runtime `AddNativeGameplayTag` |
| `bDynamicReplication` | Effective only while `FastReplication` is off | Large tag sets with differing client/server dictionaries |
| `CommonlyReplicatedTags` | Empty | List your hottest tags so they take low net indices and fit in the first bit segment |
| `NetIndexFirstBitSegment` | Project default | Size it so `CommonlyReplicatedTags` fits; tags beyond it pay one extra "more" bit plus a second segment |
| `NumBitsForContainerSize` | Project default | Set from the largest container you actually replicate |

`ShouldUseFastReplication()` and `ShouldUseDynamicReplication()` report the effective mode;
`GetNetIndexTrueBitNum()` is `ceil(log2(InvalidTagNetIndex))`, where `InvalidTagNetIndex` is the replicated tag count + 1 (`GameplayTagsManager.cpp:811`); 16 is only the value before the tag tree is built (`:347`), not a cap.
