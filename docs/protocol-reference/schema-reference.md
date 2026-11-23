# Evidence JSON schema reference

This is a data-contract index for the JSON schemas under `schemas\evidence`. These schemas describe evidence and analysis artifacts; they are not the public TCP message format. Each entry lists the exact top-level properties and indicates required properties with **required**. Nested object constraints remain normative in the schema itself. Four example JSON files are also inventoried below.

## How to read the inventory

- `ref $defs/name` means the property points to a reusable definition in the same schema; a URI reference means it reuses another evidence schema.
- An `array` entry has per-item constraints in its schema definition; the root-property table does not flatten every nested item.
- All sample identifiers and hashes in the examples below are synthetic. The excerpts are for shape recognition, not complete valid records. Schema `const` labels embedding project-tracker identifiers are shown with neutral labels; the declared field, type, and requiredness remain accurate.
- The 39 root models comprise 35 filename-titled sections and four tracker-neutral
  sections: [ABI handoff](#abi-handoff-contract-tracker-neutral-label),
  [mapping distinctness](#mapping-distinctness-contract-tracker-neutral-label),
  [post-commit binding](#post-commit-binding-contract-tracker-neutral-label), and
  [registration-pair reconciliation](#registration-pair-reconciliation-contract-tracker-neutral-label).
  Those four are inventoried below, not omitted; use the corresponding schema file's
  literal `schema` value when producing a record.

## `schemas\evidence\simsonline-adjudication-input-v1.schema.json`

**Schema title:** SimsOnline Adjudication Input v1. **Required root keys:** `schema`, `analysisIdentity`, `correlationId`, `adjudicator`, `adjudicatorVersion`, `adjudicatorGitCommit`, `inputs`, `schemasUsed`, `authorizationScope`, `unresolvedLimitations`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-adjudication-input-v1` | required |
| `analysisIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-analysis-identity-v1.schema.json` | required |
| `correlationId` | `ref $defs/correlationId` | required |
| `adjudicator` | `string` | required |
| `adjudicatorVersion` | `string` | required |
| `adjudicatorGitCommit` | `string` | required |
| `inputs` | `array` | required |
| `schemasUsed` | `array` | required |
| `authorizationScope` | `object` | required |
| `unresolvedLimitations` | `array` | required |

## `schemas\evidence\simsonline-adjudication-v1.schema.json`

**Schema title:** SimsOnline Offline Adjudication v1. **Required root keys:** `schema`, `contractVersion`, `correlationId`, `adjudicator`, `inputs`, `verdict`, `exitCode`, `reasonCodes`, `terminalEvidence`, `historicalVerdictsChanged`, `liveStageAuthorized`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-adjudication-v1` | required |
| `contractVersion` | `const simsonline-evidence-v1` | required |
| `correlationId` | `string` | required |
| `adjudicator` | `object` | required |
| `inputs` | `array` | required |
| `verdict` | `string` | required |
| `exitCode` | `integer` | required |
| `reasonCodes` | `array` | required |
| `mandatoryMilestones` | `object` | optional |
| `terminalEvidence` | `object` | required |
| `mutationFindings` | `array` | optional |
| `lifecycleFindings` | `array` | optional |
| `unresolvedQuestions` | `array` | optional |
| `historicalVerdictsChanged` | `boolean` | required |
| `liveStageAuthorized` | `boolean` | required |
| `authorizationRecord` | `string or null` | optional |
| `outputSha256` | `string or null` | optional |

## `schemas\evidence\simsonline-analysis-bindings-v1.schema.json`

**Schema title:** SimsOnline Analysis Bindings v1. **Required root keys:** `schema`, `correlationId`, `analysisIdentityArtifacts`, `bindings`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-analysis-bindings-v1` | required |
| `correlationId` | `ref $defs/correlationId` | required |
| `analysisIdentityArtifacts` | `array` | required |
| `bindings` | `array` | required |

## `schemas\evidence\simsonline-analysis-identity-v1.schema.json`

**Schema title:** SimsOnline Analysis Identity v1. **Required root keys:** `schema`, `correlationId`, `analysisRunId`, `binarySetId`, `executableSha256`, `toolchainId`, `ghidraVersion`, `ghidraDistributionSha256`, `javaIdentity`, `pythonIdentity`, `requirementsLockSha256`, `analysisProfileSha256`, `preAnalysisScriptSha256`, `exporterScriptSha256`, `repositoryCommit`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-analysis-identity-v1` | required |
| `correlationId` | `string` | required |
| `analysisRunId` | `string` | required |
| `binarySetId` | `ref $defs/sha256` | required |
| `executableSha256` | `ref $defs/sha256` | required |
| `toolchainId` | `ref $defs/sha256` | required |
| `ghidraVersion` | `const 11.2.1` | required |
| `ghidraDistributionSha256` | `ref $defs/sha256` | required |
| `javaIdentity` | `string` | required |
| `pythonIdentity` | `string` | required |
| `requirementsLockSha256` | `ref $defs/sha256` | required |
| `analysisProfileSha256` | `ref $defs/sha256` | required |
| `preAnalysisScriptSha256` | `ref $defs/sha256` | required |
| `exporterScriptSha256` | `ref $defs/sha256` | required |
| `repositoryCommit` | `string` | required |

## `schemas\evidence\simsonline-anchor-map-summary-v1.schema.json`

**Schema title:** SimsOnline  Anchor Map Summary v1. **Required root keys:** `schema`, `correlationId`, `repositoryCommit`, `binarySetId`, `managedArtifactSetId`, `toolchainId`, `normalizationVersion`, `counts`, `relationshipTypeCounts`, `subsystemCounts`, `canonicalHashes`, `mappingGapCount`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-anchor-map-summary-v1` | required |
| `correlationId` | `ref $defs/correlationId` | required |
| `repositoryCommit` | `ref $defs/sha1` | required |
| `binarySetId` | `ref $defs/sha256` | required |
| `managedArtifactSetId` | `ref $defs/artifactSetId` | required |
| `toolchainId` | `ref $defs/sha256` | required |
| `normalizationVersion` | `ref $defs/version` | required |
| `counts` | `object` | required |
| `relationshipTypeCounts` | `object` | required |
| `subsystemCounts` | `object` | required |
| `canonicalHashes` | `object` | required |
| `mappingGapCount` | `integer` | required |

## `schemas\evidence\simsonline-anchor-validation-v1.schema.json`

**Schema title:** SimsOnline Anchor Validation v1. **Required root keys:** `schema`, `analysisIdentity`, `binarySetId`, `anchorSourceLock`, `evidenceRefs`, `anchors`, `allCorroborated`, `verdict`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-anchor-validation-v1` | required |
| `analysisIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-analysis-identity-v1.schema.json` | required |
| `binarySetId` | `ref $defs/sha256` | required |
| `anchorSourceLock` | `object` | required |
| `evidenceRefs` | `array` | required |
| `anchors` | `array` | required |
| `allCorroborated` | `boolean` | required |
| `verdict` | `enum` | required |

## `schemas\evidence\simsonline-archive-manifest-v1.schema.json`

**Schema title:** SimsOnline Archive Manifest v1. **Required root keys:** `schema`, `analysisIdentity`, `correlationId`, `generatedAtUtc`, `entries`, `rootSha256`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-archive-manifest-v1` | required |
| `analysisIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-analysis-identity-v1.schema.json` | required |
| `correlationId` | `ref $defs/correlationId` | required |
| `generatedAtUtc` | `ref $defs/rfc3339` | required |
| `entries` | `array` | required |
| `rootSha256` | `ref $defs/sha256` | required |

## `schemas\evidence\simsonline-archive-seal-v1.schema.json`

**Schema title:** SimsOnline Archive Seal v1. **Required root keys:** `schema`, `analysisIdentity`, `correlationId`, `manifestSha256`, `rootSha256`, `analysisIdentityArtifactHashes`, `sealedAtUtc`, `repositoryCommit`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-archive-seal-v1` | required |
| `analysisIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-analysis-identity-v1.schema.json` | required |
| `correlationId` | `string` | required |
| `manifestSha256` | `ref $defs/sha256` | required |
| `rootSha256` | `ref $defs/sha256` | required |
| `analysisIdentityArtifactHashes` | `object` | required |
| `sealedAtUtc` | `string` | required |
| `repositoryCommit` | `string` | required |

## `schemas\evidence\simsonline-binary-inventory-v1.schema.json`

**Schema title:** SimsOnline Binary Inventory v1. **Required root keys:** `schema`, `analysisIdentity`, `binarySet`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-binary-inventory-v1` | required |
| `analysisIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-analysis-identity-v1.schema.json` | required |
| `binarySet` | `object` | required |

## `schemas\evidence\simsonline-campaign-execution-report-v1.schema.json`

**Schema title:** SimsOnline Campaign Execution Report v1. **Required root keys:** `schema`, `analysisIdentity`, `correlationId`, `generatedAtUtc`, `operations`, `passed`, `failed`, `skipped`, `result`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-campaign-execution-report-v1` | required |
| `analysisIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-analysis-identity-v1.schema.json` | required |
| `correlationId` | `ref $defs/correlationId` | required |
| `generatedAtUtc` | `ref $defs/rfc3339` | required |
| `operations` | `array` | required |
| `passed` | `integer` | required |
| `failed` | `integer` | required |
| `skipped` | `integer` | required |
| `result` | `enum` | required |

## `schemas\evidence\simsonline-campaign-manifest-v1.schema.json`

**Schema title:** SimsOnline Campaign Evidence Manifest v1. **Required root keys:** `schema`, `contractVersion`, `correlationId`, `ownerIssues`, `scenario`, `timestamps`, `repository`, `binarySet`, `producers`, `authorization`, `artifacts`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-campaign-manifest-v1` | required |
| `contractVersion` | `const simsonline-evidence-v1` | required |
| `correlationId` | `ref $defs/correlationId` | required |
| `ownerIssues` | `array` | required |
| `scenario` | `string` | required |
| `timestamps` | `object` | required |
| `repository` | `object` | required |
| `binarySet` | `object` | required |
| `producers` | `array` | required |
| `authorization` | `object` | required |
| `absentProducers` | `array` | optional |
| `sourceArchives` | `array` | optional |
| `consumedMarkers` | `array` | optional |
| `artifacts` | `array` | required |
| `integrity` | `object` | optional |

## `schemas\evidence\simsonline-dynamic-confirmation-queue-v1.schema.json`

**Schema title:** SimsOnline  Dynamic Confirmation Queue v1. **Required root keys:** `schema`, `correlationId`, `repositoryCommit`, `binarySetId`, `managedArtifactSetId`, `toolchainId`, `normalizationVersion`, `traceAuthorized`, `entries`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-dynamic-confirmation-queue-v1` | required |
| `correlationId` | `ref $defs/correlationId` | required |
| `repositoryCommit` | `ref $defs/sha1` | required |
| `binarySetId` | `ref $defs/sha256` | required |
| `managedArtifactSetId` | `ref $defs/artifactSetId` | required |
| `toolchainId` | `ref $defs/sha256` | required |
| `normalizationVersion` | `ref $defs/version` | required |
| `traceAuthorized` | `boolean` | required |
| `entries` | `array` | required |

## `schemas\evidence\simsonline-effective-analysis-profile-v1.schema.json`

**Schema title:** SimsOnline Effective Analysis Profile v1. **Required root keys:** `schema`, `profileSha256`, `language`, `compilerSpecId`, `timeoutSeconds`, `maxCpu`, `enabledAnalyzers`, `disabledAnalyzers`, `analyzerOptions`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-effective-analysis-profile-v1` | required |
| `analysisIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-analysis-identity-v1.schema.json` | optional |
| `syntheticSmokeIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-synthetic-smoke-identity-v1.schema.json` | optional |
| `profileSha256` | `ref $defs/sha256` | required |
| `language` | `const x86:LE:32:default` | required |
| `compilerSpecId` | `const windows` | required |
| `timeoutSeconds` | `integer` | required |
| `maxCpu` | `integer` | required |
| `enabledAnalyzers` | `array` | required |
| `disabledAnalyzers` | `array` | required |
| `analyzerOptions` | `object` | required |

## `schemas\evidence\simsonline-event-v1.schema.json`

**Schema title:** SimsOnline Managed/Native Event Envelope v1. **Required root keys:** `schema`, `correlationId`, `source`, `stream`, `eventType`, `sequence`, `observedAtUtc`, `clockDomain`, `payload`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-event-v1` | required |
| `correlationId` | `string` | required |
| `source` | `enum` | required |
| `stream` | `string` | required |
| `eventType` | `string` | required |
| `sequence` | `integer` | required |
| `observedAtUtc` | `string` | required |
| `clockDomain` | `string` | required |
| `processId` | `integer or null` | optional |
| `threadId` | `integer or null` | optional |
| `monotonicNs` | `integer or null` | optional |
| `simTick` | `integer or null` | optional |
| `clockOffsetEstimateNs` | `integer or null` | optional |
| `clockUncertaintyNs` | `integer or null` | optional |
| `orderingConfidence` | `enum` | optional |
| `payload` | `object` | required |
| `resourceStates` | `object` | optional |
| `queueRoles` | `object` | optional |
| `symbolRefs` | `array` | optional |
| `evidenceRefs` | `array` | optional |

## `schemas\evidence\simsonline-execution-source-manifest-v1.schema.json`

**Schema title:** SimsOnline Execution Source Manifest v1. **Required root keys:** `schema`, `analysisIdentity`, `correlationId`, `generatedAtUtc`, `baseCommit`, `dirty`, `allowedDirtyPaths`, `entries`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-execution-source-manifest-v1` | required |
| `analysisIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-analysis-identity-v1.schema.json` | required |
| `correlationId` | `ref $defs/correlationId` | required |
| `generatedAtUtc` | `ref $defs/rfc3339` | required |
| `baseCommit` | `string` | required |
| `dirty` | `boolean` | required |
| `allowedDirtyPaths` | `array` | required |
| `entries` | `array` | required |

## `schemas\evidence\simsonline-gate-report-v1.schema.json`

**Schema title:** SimsOnline Mandatory Gate Report v1. **Required root keys:** `schema`, `analysisIdentity`, `correlationId`, `generatedAtUtc`, `stageResult`, `gates`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-gate-report-v1` | required |
| `analysisIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-analysis-identity-v1.schema.json` | required |
| `correlationId` | `ref $defs/correlationId` | required |
| `generatedAtUtc` | `ref $defs/rfc3339` | required |
| `stageResult` | `enum` | required |
| `gates` | `object` | required |

## `schemas\evidence\simsonline-ghidra-analyzer-inventory-v1.schema.json`

**Schema title:** SimsOnline Ghidra Analyzer Inventory v1. **Required root keys:** `schema`, `profileSha256`, `preAnalysisScriptSha256`, `exporterScriptSha256`, `observedOptions`, `requested`, `effective`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-ghidra-analyzer-inventory-v1` | required |
| `syntheticSmokeIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-synthetic-smoke-identity-v1.schema.json` | optional |
| `analysisIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-analysis-identity-v1.schema.json` | optional |
| `profileSha256` | `ref $defs/sha256` | required |
| `preAnalysisScriptSha256` | `ref $defs/sha256` | required |
| `exporterScriptSha256` | `ref $defs/sha256` | required |
| `observedOptions` | `array` | required |
| `requested` | `ref $defs/analyzerState` | required |
| `effective` | `ref $defs/analyzerState` | required |

## `schemas\evidence\simsonline-managed-artifact-inventory-v1.schema.json`

**Schema title:** SimsOnline  Managed Artifact Inventory v1. **Required root keys:** `schema`, `correlationId`, `repositoryCommit`, `binarySetId`, `managedArtifactSetId`, `toolchainId`, `normalizationVersion`, `artifacts`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-managed-artifact-inventory-v1` | required |
| `correlationId` | `ref $defs/correlationId` | required |
| `repositoryCommit` | `ref $defs/sha1` | required |
| `binarySetId` | `ref $defs/sha256` | required |
| `managedArtifactSetId` | `ref $defs/artifactSetId` | required |
| `toolchainId` | `ref $defs/sha256` | required |
| `normalizationVersion` | `ref $defs/version` | required |
| `artifacts` | `array` | required |

## `schemas\evidence\simsonline-managed-boundary-v1.schema.json`

**Schema title:** SimsOnline  Managed Boundary v1. **Required root keys:** `schema`, `correlationId`, `repositoryCommit`, `binarySetId`, `managedArtifactSetId`, `toolchainId`, `normalizationVersion`, `boundaryId`, `managedArtifactId`, `metadataToken`, `declaringType`, `managedMember`, `genericArity`, `normalizedSignature`, `returnType`, `parameterTypes`, `isStatic`, `boundaryKind`, `pinvoke`, `internalCall`, `subsystemHints`, `evidenceRefs`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-managed-boundary-v1` | required |
| `correlationId` | `ref $defs/correlationId` | required |
| `repositoryCommit` | `ref $defs/sha1` | required |
| `binarySetId` | `ref $defs/sha256` | required |
| `managedArtifactSetId` | `ref $defs/artifactSetId` | required |
| `toolchainId` | `ref $defs/sha256` | required |
| `normalizationVersion` | `ref $defs/version` | required |
| `boundaryId` | `ref $defs/boundaryId` | required |
| `managedArtifactId` | `ref $defs/artifactId` | required |
| `metadataToken` | `string` | required |
| `declaringType` | `string` | required |
| `managedMember` | `string` | required |
| `genericArity` | `integer` | required |
| `normalizedSignature` | `string` | required |
| `returnType` | `string` | required |
| `parameterTypes` | `array` | required |
| `isStatic` | `boolean` | required |
| `boundaryKind` | `enum` | required |
| `pinvoke` | `oneOf (null, object)` | required |
| `internalCall` | `oneOf (null, object)` | required |
| `subsystemHints` | `array` | required |
| `evidenceRefs` | `array` | required |

## `schemas\evidence\simsonline-managed-call-edge-v1.schema.json`

**Schema title:** SimsOnline  Managed Call Edge v1. **Required root keys:** `schema`, `correlationId`, `repositoryCommit`, `binarySetId`, `managedArtifactSetId`, `toolchainId`, `normalizationVersion`, `edgeId`, `callerManagedId`, `calleeBoundaryId`, `managedArtifactId`, `ilOperationKind`, `callCount`, `evidenceRefs`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-managed-call-edge-v1` | required |
| `correlationId` | `ref $defs/correlationId` | required |
| `repositoryCommit` | `ref $defs/sha1` | required |
| `binarySetId` | `ref $defs/sha256` | required |
| `managedArtifactSetId` | `ref $defs/artifactSetId` | required |
| `toolchainId` | `ref $defs/sha256` | required |
| `normalizationVersion` | `ref $defs/version` | required |
| `edgeId` | `string` | required |
| `callerManagedId` | `string` | required |
| `calleeBoundaryId` | `ref $defs/boundaryId` | required |
| `managedArtifactId` | `ref $defs/artifactId` | required |
| `ilOperationKind` | `enum` | required |
| `callCount` | `integer` | required |
| `evidenceRefs` | `array` | required |

## `schemas\evidence\simsonline-managed-native-mapping-v1.schema.json`

**Schema title:** SimsOnline  Managed Native Mapping v1. **Required root keys:** `schema`, `correlationId`, `repositoryCommit`, `binarySetId`, `managedArtifactSetId`, `toolchainId`, `normalizationVersion`, `mappingId`, `managedBoundaryId`, `nativeSymbolId`, `nativeLocator`, `relationshipType`, `verificationState`, `confidence`, `subsystem`, `provenance`, `evidenceRefs`, `unresolvedReason`, `conflictReason`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-managed-native-mapping-v1` | required |
| `correlationId` | `ref $defs/correlationId` | required |
| `repositoryCommit` | `ref $defs/sha1` | required |
| `binarySetId` | `ref $defs/sha256` | required |
| `managedArtifactSetId` | `ref $defs/artifactSetId` | required |
| `toolchainId` | `ref $defs/sha256` | required |
| `normalizationVersion` | `ref $defs/version` | required |
| `mappingId` | `string` | required |
| `managedBoundaryId` | `ref $defs/boundaryId` | required |
| `nativeSymbolId` | `string or null` | required |
| `nativeLocator` | `oneOf (null, object)` | required |
| `relationshipType` | `enum` | required |
| `verificationState` | `enum` | required |
| `confidence` | `object` | required |
| `subsystem` | `ref $defs/subsystem` | required |
| `provenance` | `array` | required |
| `evidenceRefs` | `array` | required |
| `unresolvedReason` | `string or null` | required |
| `conflictReason` | `string or null` | required |

## `schemas\evidence\simsonline-native-registration-v1.schema.json`

**Schema title:** SimsOnline  Native Registration v1. **Required root keys:** `schema`, `correlationId`, `repositoryCommit`, `binarySetId`, `managedArtifactSetId`, `toolchainId`, `normalizationVersion`, `registrationId`, `recoveryState`, `managedRegistrationName`, `registrationNameCandidates`, `managedBoundaryIds`, `registrationCallsiteRva`, `registrationFunctionRva`, `managedNameStringRva`, `nativeFunctionRva`, `containingFunctionRva`, `wrapperSymbolId`, `argumentRecoveryState`, `evidenceRefs`, `failureReason`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-native-registration-v1` | required |
| `correlationId` | `ref $defs/correlationId` | required |
| `repositoryCommit` | `ref $defs/sha1` | required |
| `binarySetId` | `ref $defs/sha256` | required |
| `managedArtifactSetId` | `ref $defs/artifactSetId` | required |
| `toolchainId` | `ref $defs/sha256` | required |
| `normalizationVersion` | `ref $defs/version` | required |
| `registrationId` | `string` | required |
| `recoveryState` | `enum` | required |
| `managedRegistrationName` | `string or null` | required |
| `registrationNameCandidates` | `array` | required |
| `managedBoundaryIds` | `array` | required |
| `registrationCallsiteRva` | `ref $defs/nullableRva` | required |
| `registrationFunctionRva` | `ref $defs/nullableRva` | required |
| `managedNameStringRva` | `ref $defs/nullableRva` | required |
| `nativeFunctionRva` | `ref $defs/nullableRva` | required |
| `containingFunctionRva` | `ref $defs/nullableRva` | required |
| `wrapperSymbolId` | `string or null` | required |
| `argumentRecoveryState` | `enum` | required |
| `evidenceRefs` | `array` | required |
| `failureReason` | `string or null` | required |

## `schemas\evidence\simsonline-normalized-static-export-v1.schema.json`

**Schema title:** SimsOnline Normalized Static Export v1. **Required root keys:** `_normalization`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-normalized-static-export-v1` | optional |
| `_normalization` | `object` | required |

## `schemas\evidence\simsonline-preflight-report-v1.schema.json`

**Schema title:** SimsOnline Preflight Report v1. **Required root keys:** `schema`, `analysisIdentity`, `correlationId`, `generatedAtUtc`, `checks`, `passed`, `failed`, `skipped`, `verdict`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-preflight-report-v1` | required |
| `analysisIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-analysis-identity-v1.schema.json` | required |
| `correlationId` | `ref $defs/correlationId` | required |
| `generatedAtUtc` | `ref $defs/rfc3339` | required |
| `checks` | `array` | required |
| `passed` | `integer` | required |
| `failed` | `integer` | required |
| `skipped` | `integer` | required |
| `verdict` | `enum` | required |

## `schemas\evidence\simsonline-reproducibility-comparison-v1.schema.json`

**Schema title:** SimsOnline Reproducibility Comparison v1. **Required root keys:** `schema`, `analysisIdentity`, `correlationId`, `run1AnalysisRunId`, `run2AnalysisRunId`, `run1Hash`, `run2Hash`, `canonicalHashesEqual`, `identitiesMatch`, `differences`, `verdict`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-reproducibility-comparison-v1` | required |
| `analysisIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-analysis-identity-v1.schema.json` | required |
| `correlationId` | `ref $defs/correlationId` | required |
| `run1AnalysisRunId` | `ref $defs/runId` | required |
| `run2AnalysisRunId` | `ref $defs/runId` | required |
| `run1Hash` | `ref $defs/sha256` | required |
| `run2Hash` | `ref $defs/sha256` | required |
| `canonicalHashesEqual` | `boolean` | required |
| `identitiesMatch` | `boolean` | required |
| `differences` | `array` | required |
| `verdict` | `enum` | required |

## `schemas\evidence\simsonline-safety-report-v1.schema.json`

**Schema title:** SimsOnline Safety Report v1. **Required root keys:** `schema`, `analysisIdentity`, `correlationId`, `generatedAtUtc`, `passed`, `scannedScopes`, `violations`, `exemptions`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-safety-report-v1` | required |
| `analysisIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-analysis-identity-v1.schema.json` | required |
| `correlationId` | `ref $defs/correlationId` | required |
| `generatedAtUtc` | `ref $defs/rfc3339` | required |
| `passed` | `boolean` | required |
| `scannedScopes` | `array` | required |
| `violations` | `array` | required |
| `exemptions` | `array` | required |

## `schemas\evidence\simsonline-schema-validation-report-v1.schema.json`

**Schema title:** SimsOnline Schema Validation Report v1. **Required root keys:** `schema`, `analysisIdentity`, `correlationId`, `generatedAtUtc`, `passed`, `validations`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-schema-validation-report-v1` | required |
| `analysisIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-analysis-identity-v1.schema.json` | required |
| `correlationId` | `ref $defs/correlationId` | required |
| `generatedAtUtc` | `ref $defs/rfc3339` | required |
| `passed` | `boolean` | required |
| `validations` | `array` | required |

## `schemas\evidence\simsonline-signature-scan-v1.schema.json`

**Schema title:** SimsOnline Signature Scan v1. **Required root keys:** `schema`, `analysisIdentity`, `binarySetId`, `executable`, `anchorSourceLock`, `sourceBlob`, `anchors`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-signature-scan-v1` | required |
| `analysisIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-analysis-identity-v1.schema.json` | required |
| `binarySetId` | `ref $defs/sha256` | required |
| `executable` | `ref $defs/fileRef` | required |
| `anchorSourceLock` | `ref $defs/fileRef` | required |
| `sourceBlob` | `object` | required |
| `anchors` | `array` | required |

## ABI handoff contract (tracker-neutral label)

**Schema title:** SimsOnline  Static ABI Handoff v1. **Required root keys:** `schema`, `staticOnly`, `traceAuthorized`, `primary`, `alternate`, `recommendationReason`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const [ABI handoff schema identifier]` | required |
| `staticOnly` | `const True` | required |
| `traceAuthorized` | `const False` | required |
| `primary` | `ref $defs/candidate` | required |
| `alternate` | `ref $defs/candidate` | required |
| `recommendationReason` | `string` | required |

## Mapping distinctness contract (tracker-neutral label)

**Schema title:** SimsOnline  Mapping Distinctness v1. **Required root keys:** `schema`, `mappingRecordCount`, `corroboratedRecordCount`, `corroboratedDistinctPairCount`, `pairIdentityFields`, `pairs`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const [mapping distinctness schema identifier]` | required |
| `mappingRecordCount` | `integer` | required |
| `corroboratedRecordCount` | `integer` | required |
| `corroboratedDistinctPairCount` | `integer` | required |
| `pairIdentityFields` | `const ['managedBoundaryId', 'nativeModuleSha256', 'nativeTargetRva']` | required |
| `pairs` | `array` | required |

## Post-commit binding contract (tracker-neutral label)

**Schema title:** SimsOnline  Post-Commit Binding v1. **Required root keys:** `schema`, `implementationCommit`, `implementationParent`, `implementationTree`, `campaignCorrelationId`, `campaignRepositoryCommit`, `campaignWasPreCommit`, `executionSourceManifest`, `executionSourceManifestSha256`, `campaignSourceCount`, `campaignSourceMatchCount`, `campaignSourceMismatchCount`, `allCampaignSourcesMatchCommit`, `archiveRootSha256`, `manifestSha256`, `externalSealSha256`, `formalTechnicalVerdict`, `bindingVerdict`, `historicalArchiveModified`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const [post-commit binding schema identifier]` | required |
| `implementationCommit` | `ref $defs/sha1` | required |
| `implementationParent` | `ref $defs/sha1` | required |
| `implementationTree` | `ref $defs/sha1` | required |
| `campaignCorrelationId` | `string` | required |
| `campaignRepositoryCommit` | `ref $defs/sha1` | required |
| `campaignWasPreCommit` | `boolean` | required |
| `executionSourceManifest` | `string` | required |
| `executionSourceManifestSha256` | `ref $defs/sha256` | required |
| `campaignSourceCount` | `integer` | required |
| `campaignSourceMatchCount` | `integer` | required |
| `campaignSourceMismatchCount` | `integer` | required |
| `allCampaignSourcesMatchCommit` | `boolean` | required |
| `archiveRootSha256` | `ref $defs/sha256` | required |
| `manifestSha256` | `ref $defs/sha256` | required |
| `externalSealSha256` | `ref $defs/sha256` | required |
| `formalTechnicalVerdict` | `const PASS_STATIC_ANCHOR_MAP` | required |
| `bindingVerdict` | `enum` | required |
| `historicalArchiveModified` | `const False` | required |

## Registration-pair reconciliation contract (tracker-neutral label)

**Schema title:** SimsOnline  Registration Pair Reconciliation v1. **Required root keys:** `schema`, `callsiteCount`, `exactPairCount`, `indirectUnresolvedCount`, `exactPairDispositionCounts`, `reconciledExactPairCount`, `complete`, `dispositions`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const [registration-pair reconciliation schema identifier]` | required |
| `callsiteCount` | `integer` | required |
| `exactPairCount` | `integer` | required |
| `indirectUnresolvedCount` | `integer` | required |
| `exactPairDispositionCounts` | `ref $defs/counts` | required |
| `reconciledExactPairCount` | `integer` | required |
| `complete` | `boolean` | required |
| `dispositions` | `array` | required |

## `schemas\evidence\simsonline-static-export-v1.schema.json`

**Schema title:** SimsOnline Bounded Static Export v1. **Required root keys:** `schema`, `stage`, `programName`, `imageBase`, `minAddress`, `maxAddress`, `effectiveAnalysisProfile`, `summary`, `sections`, `imports`, `exports`, `functions`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `enum` | required |
| `stage` | `const analysis-complete` | required |
| `analysisIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-analysis-identity-v1.schema.json` | optional |
| `syntheticSmokeIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-synthetic-smoke-identity-v1.schema.json` | optional |
| `programName` | `string` | required |
| `imageBase` | `ref $defs/address` | required |
| `minAddress` | `ref $defs/address` | required |
| `maxAddress` | `ref $defs/address` | required |
| `effectiveAnalysisProfile` | `ref $defs/profile` | required |
| `summary` | `ref $defs/summary` | required |
| `sections` | `array` | required |
| `imports` | `array` | required |
| `exports` | `array` | required |
| `functions` | `array` | required |
| `evidenceRefs` | `array` | optional |
| `artifacts` | `array` | optional |
| `modules` | `array` | optional |
| `analyzerOptions` | `array` | optional |
| `_normalization` | `ref $defs/normalization` | optional |

## `schemas\evidence\simsonline-subsystem-taxonomy-v1.schema.json`

**Schema title:** SimsOnline  Subsystem Taxonomy v1. **Required root keys:** `schema`, `correlationId`, `repositoryCommit`, `binarySetId`, `managedArtifactSetId`, `toolchainId`, `normalizationVersion`, `taxonomyVersion`, `subsystems`, `classificationRules`, `reviewedOverrides`, `assignments`, `aiSuggestionArtifact`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-subsystem-taxonomy-v1` | required |
| `correlationId` | `ref $defs/correlationId` | required |
| `repositoryCommit` | `ref $defs/sha1` | required |
| `binarySetId` | `ref $defs/sha256` | required |
| `managedArtifactSetId` | `ref $defs/artifactSetId` | required |
| `toolchainId` | `ref $defs/sha256` | required |
| `normalizationVersion` | `ref $defs/version` | required |
| `taxonomyVersion` | `const [subsystem taxonomy version identifier]` | required |
| `subsystems` | `array` | required |
| `classificationRules` | `array` | required |
| `reviewedOverrides` | `array` | required |
| `assignments` | `array` | required |
| `aiSuggestionArtifact` | `oneOf (null, ref $defs/evidenceRef)` | required |

## `schemas\evidence\simsonline-symbol-catalog-v1.schema.json`

**Schema title:** SimsOnline Native Symbol Catalog v1. **Required root keys:** `schema`, `contractVersion`, `binarySetId`, `generatedAtUtc`, `symbols`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-symbol-catalog-v1` | required |
| `contractVersion` | `const simsonline-evidence-v1` | required |
| `binarySetId` | `string` | required |
| `generatedAtUtc` | `string` | required |
| `generator` | `object` | optional |
| `symbols` | `array` | required |

## `schemas\evidence\simsonline-synthetic-headless-smoke-v1.schema.json`

**Schema title:** SimsOnline Synthetic Headless Smoke v1. **Required root keys:** `schema`, `stage`, `syntheticSmokeIdentity`, `input`, `analyzerInventory`, `effectiveProfile`, `rawExport`, `receipt`, `passed`, `issues`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-synthetic-headless-smoke-v1` | required |
| `stage` | `const analysis-complete` | required |
| `syntheticSmokeIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-synthetic-smoke-identity-v1.schema.json` | required |
| `input` | `object` | required |
| `analyzerInventory` | `ref $defs/artifactRef` | required |
| `effectiveProfile` | `ref $defs/artifactRef` | required |
| `rawExport` | `ref $defs/artifactRef` | required |
| `receipt` | `ref $defs/artifactRef` | required |
| `execution` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-test-execution-report-v1.schema.json$defs/command` | optional |
| `passed` | `boolean` | required |
| `issues` | `array` | required |

## `schemas\evidence\simsonline-synthetic-smoke-identity-v1.schema.json`

**Schema title:** SimsOnline Synthetic Smoke Identity v1. **Required root keys:** `schema`, `correlationId`, `syntheticPeSha256`, `ghidraVersion`, `ghidraDistributionSha256`, `javaIdentity`, `pythonIdentity`, `requirementsLockSha256`, `analysisProfileSha256`, `preAnalysisScriptSha256`, `exporterScriptSha256`, `repositoryCommit`, `toolchainId`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-synthetic-smoke-identity-v1` | required |
| `correlationId` | `string` | required |
| `syntheticPeSha256` | `ref $defs/sha256` | required |
| `ghidraVersion` | `const 11.2.1` | required |
| `ghidraDistributionSha256` | `ref $defs/sha256` | required |
| `javaIdentity` | `string` | required |
| `pythonIdentity` | `string` | required |
| `requirementsLockSha256` | `ref $defs/sha256` | required |
| `analysisProfileSha256` | `ref $defs/sha256` | required |
| `preAnalysisScriptSha256` | `ref $defs/sha256` | required |
| `exporterScriptSha256` | `ref $defs/sha256` | required |
| `repositoryCommit` | `string` | required |
| `toolchainId` | `ref $defs/sha256` | required |

## `schemas\evidence\simsonline-test-execution-report-v1.schema.json`

**Schema title:** SimsOnline Test Execution Report v1. **Required root keys:** `schema`, `analysisIdentity`, `correlationId`, `generatedAtUtc`, `commands`, `passed`, `failed`, `skipped`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-test-execution-report-v1` | required |
| `analysisIdentity` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-analysis-identity-v1.schema.json` | required |
| `correlationId` | `ref $defs/correlationId` | required |
| `generatedAtUtc` | `ref $defs/rfc3339` | required |
| `commands` | `array` | required |
| `passed` | `integer` | required |
| `failed` | `integer` | required |
| `skipped` | `integer` | required |

## `schemas\evidence\simsonline-toolchain-verification-v1.schema.json`

**Schema title:** SimsOnline Toolchain Verification v1. **Required root keys:** `schema`, `stage`, `ghidraVersion`, `ghidraDistributionSha256`, `lockSyntaxVerified`, `archiveVerified`, `installVerified`, `hostEnvironmentVerified`, `javaDistributionSha256`, `javaArchiveVerified`, `javaInstallVerified`, `javaVerified`, `pythonDependenciesVerified`, `formatValidationVerified`, `javaIdentity`, `pythonIdentity`, `requirementsLockSha256`, `analysisProfileSha256`, `preAnalysisScriptSha256`, `exporterScriptSha256`, `repositoryCommit`, `verified`, `issues`.

| Property | JSON Schema type / reference | Presence |
|---|---|---|
| `schema` | `const simsonline-toolchain-verification-v1` | required |
| `stage` | `const toolchain-verification-complete` | required |
| `ghidraVersion` | `const 11.2.1` | required |
| `ghidraDistributionSha256` | `ref $defs/sha256` | required |
| `lockSyntaxVerified` | `boolean` | required |
| `archiveVerified` | `boolean` | required |
| `installVerified` | `boolean` | required |
| `hostEnvironmentVerified` | `boolean` | required |
| `javaDistributionSha256` | `ref $defs/sha256` | required |
| `javaArchiveVerified` | `boolean` | required |
| `javaInstallVerified` | `boolean` | required |
| `javaVerified` | `boolean` | required |
| `pythonDependenciesVerified` | `boolean` | required |
| `formatValidationVerified` | `boolean` | required |
| `javaIdentity` | `string` | required |
| `pythonIdentity` | `string` | required |
| `requirementsLockSha256` | `ref $defs/sha256` | required |
| `analysisProfileSha256` | `ref $defs/sha256` | required |
| `preAnalysisScriptSha256` | `ref $defs/sha256` | required |
| `exporterScriptSha256` | `ref $defs/sha256` | required |
| `repositoryCommit` | `string` | required |
| `execution` | `ref https://schemas.obsidiannetwork.au/simsonline/evidence/simsonline-test-execution-report-v1.schema.json$defs/command` | optional |
| `verified` | `boolean` | required |
| `issues` | `array` | required |

## Example payloads

The repository includes four static-analysis examples. The snippets below preserve the contract shape while replacing project-specific identifiers and values.

### Event envelope

`examples\static-analysis\valid-event.json` demonstrates an event record: `schema`, `correlationId`, `source`, `stream`, `eventType`, `sequence`, `observedAtUtc`, `clockDomain`, and `payload` are the core fields. Optional process/thread/clock metadata and resource/evidence references extend the same envelope.

```json
{"schema":"simsonline-event-v1","correlationId":"so1-example-run-20260102T030405Z-01234567","source":"system","stream":"system","eventType":"analysis.completed","sequence":1,"observedAtUtc":"2026-01-02T03:04:05Z","clockDomain":"system-process","payload":{"result":"pass"}}
```

### Adjudication result

`examples\static-analysis\valid-adjudication.json` shows the result side of the contract: verdict and exit status sit beside input digests, terminal evidence, and authorization facts. This excerpt omits nested evidence details.

```json
{"schema":"simsonline-adjudication-v1","contractVersion":"simsonline-evidence-v1","correlationId":"so1-example-run-20260102T030405Z-01234567","adjudicator":{"name":"offline-adjudicator","version":"1.0.0","gitCommitSha":"0000000000000000000000000000000000000000","adjudicatedAtUtc":"2026-01-02T03:04:05Z"},"inputs":[],"verdict":"PASS_STATIC_FOUNDATION","exitCode":0,"reasonCodes":[],"terminalEvidence":{"resourceStates":{},"queueRoles":{},"unknownFields":[]},"historicalVerdictsChanged":false,"liveStageAuthorized":false}
```

### Campaign manifest

`examples\static-analysis\valid-campaign-manifest.json` demonstrates provenance and authorization metadata. A manifest records repository identity, binary inventory, producers, artifacts, and the explicit authorization booleans; those booleans should not be inferred from the presence of evidence.

```json
{"schema":"simsonline-campaign-manifest-v1","contractVersion":"simsonline-evidence-v1","correlationId":"so1-example-run-20260102T030405Z-01234567","scenario":"static-analysis-example","timestamps":{"createdAtUtc":"2026-01-02T03:04:05Z"},"repository":{"name":"example/project","branch":"release","commitSha":"0000000000000000000000000000000000000000","dirty":false},"producers":[],"authorization":{"liveExecutionAuthorized":false,"queueMutationAuthorized":false,"nativeInjectionAuthorized":false,"clientParityAuthorized":false},"artifacts":[]}
```

### Symbol catalog

`examples\static-analysis\valid-symbol-catalog.json` illustrates a catalog-level envelope around symbol records. The excerpt intentionally uses a synthetic symbol and locator rather than real game addresses or symbol data.

```json
{"schema":"simsonline-symbol-catalog-v1","contractVersion":"simsonline-evidence-v1","binarySetId":"0000000000000000000000000000000000000000000000000000000000000000","generatedAtUtc":"2026-01-02T03:04:05Z","symbols":[{"symbolId":"example:method:0001","moduleName":"Example.exe","moduleSha256":"0000000000000000000000000000000000000000000000000000000000000000","proposedName":"ExampleMethod","verificationState":"CANDIDATE","confidence":0.5,"locator":{"rva":"0x00001000"},"provenance":["SYNTHETIC"],"evidenceRefs":[]}]}
```

The corresponding checked-in example payloads are `schemas\evidence\examples\static-analysis\valid-adjudication.json`, `valid-campaign-manifest.json`, `valid-event.json`, and `valid-symbol-catalog.json`. The snippets above are redacted illustrations, not copies of those files.

## Contract boundaries

The evidence family separates identity and toolchain attestations from analysis output: an analysis identity pins executable and toolchain digests; static exports and managed/native mappings carry their own evidence references; campaign, test, gate, safety, and adjudication reports express outcomes. A report that says a stage passed does not itself authorize a later live stage: the authorization booleans are explicit fields and must be interpreted according to their schema.

Related pages: [protocol wire formats](wire-formats.md), [bridge diagnostics](bridge-diagnostics.md), [latency and verification](latency-and-tests.md).
