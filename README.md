gh repo clone cyrillofrancisco30-lgtm/arm-ttk
cd arm-ttk
mkdir -p xa-trust/tests/jcs
git checkout -b xa-trust-implementation


#!/usr/bin/env python3
"""JCS Evidence Producer for XA-TRUST / ECM (RFC 8785 Conformance)"""

import json
import sys
import os
import hashlib
import base64
from datetime import datetime, timezone
import subprocess

try:
    import rfc8785
except ImportError:
    print("[ERROR] Required library 'rfc8785' not found. Install via pip install rfc8785", file=sys.stderr)
    sys.exit(1)

EXECUTOR_VERSION = "1.0.0"
VECTORS_PATH = "xa-trust/tests/jcs/vectors.json"

def get_sha256(data: bytes) -> str:
    return hashlib.sha256(data).hexdigest()

def get_git_head() -> str:
    try:
        return subprocess.check_output(['git', 'rev-parse', 'HEAD']).decode('utf-8').strip()
    except Exception:
        return "UNKNOWN_COMMIT"

def produce_evidence() -> bool:
    if not os.path.exists(VECTORS_PATH):
        print(f"[ERROR] Vectors file not found at: {VECTORS_PATH}", file=sys.stderr)
        sys.exit(1)

    with open(VECTORS_PATH, "r", encoding="utf-8") as f:
        data = json.load(f)

    suite_name = data.get("suite", "Unknown Suite")
    vectors = data.get("vectors", [])

    print(f"=== Producing Evidence for {suite_name} ===")

    metadata = {
        "repository": "cyrillofrancisco30-lgtm/arm-ttk",
        "head_sha": get_git_head(),
        "workflow_run_id": os.environ.get("GITHUB_RUN_ID", "LOCAL_EXECUTION"),
        "timestamp": datetime.now(timezone.utc).isoformat().replace("+00:00", "Z"),
        "executor_version": EXECUTOR_VERSION,
        "environment": {
            "python_version": sys.version.split()[0],
            "rfc8785_version": "installed"
        }
    }

    artifacts = []
    with open(VECTORS_PATH, "rb") as f:
        content = f.read()
        artifacts.append({
            "path": VECTORS_PATH,
            "sha256": hashlib.sha256(content).hexdigest(),
            "size_bytes": len(content)
        })

    execution_results = []
    all_passed = True

    for vec in vectors:
        vid = vec.get("id")
        raw_input = vec.get("input")
        expected_canonical = vec.get("expected_canonical")

        try:
            actual_bytes = rfc8785.dumps(raw_input)
            if isinstance(actual_bytes, str):
                actual_bytes = actual_bytes.encode('utf-8')

            expected_bytes = expected_canonical.encode('utf-8') if isinstance(expected_canonical, str) else expected_canonical

            res_status = "PASS" if actual_bytes == expected_bytes else "FAIL"
            if res_status == "FAIL":
                all_passed = False

            execution_results.append({
                "vector_id": vid,
                "input_sha256": get_sha256(actual_bytes),
                "expected_sha256": get_sha256(expected_bytes),
                "actual_sha256": get_sha256(actual_bytes),
                "actual_bytes_base64": base64.b64encode(actual_bytes).decode('utf-8'),
                "result": res_status
            })
            print(f"[{res_status}] {vid}")

        except Exception as e:
            print(f"[ERROR] {vid}: {e}")
            execution_results.append({
                "vector_id": vid,
                "input_sha256": "ERROR",
                "expected_sha256": "ERROR",
                "actual_sha256": "ERROR",
                "actual_bytes_base64": "",
                "result": "ERROR"
            })
            all_passed = False

    os.makedirs("xa-trust/evidence", exist_ok=True)

    with open("xa-trust/evidence/execution_metadata.json", "w") as f:
        json.dump(metadata, f, indent=2)

    with open("xa-trust/evidence/artifact_hashes.json", "w") as f:
        json.dump({"artifacts": artifacts}, f, indent=2)

    with open("xa-trust/evidence/test_execution.json", "w") as f:
        json.dump({"suite": suite_name, "results": execution_results}, f, indent=2)

    print("\n=== Evidence Production Complete ===")
    return all_passed

if __name__ == "__main__":
    success = produce_evidence()
    sys.exit(0 if success else 1)


{
  "suite": "JCS RFC 8785 Vectors - XA-TRUST",
  "vectors": [
    {
      "id": "vec-001",
      "input": {"b": 2, "a": 1},
      "expected_canonical": "{\"a\":1,\"b\":2}"
    },
    {
      "id": "vec-002",
      "input": {"claim_id": "ARM_TTK_TEST_RESULT", "status": "OBSERVED"},
      "expected_canonical": "{\"claim_id\":\"ARM_TTK_TEST_RESULT\",\"status\":\"OBSERVED\"}"
    }
  ]
}



#!/usr/bin/env python3
"""XA-TRUST Collector - E3/E4 for arm-ttk via GitHub API"""
import json
import os
import sys
import urllib.request

REPO = "cyrillofrancisco30-lgtm/arm-ttk"
RUN_ID = "37121498615"
TOKEN = os.environ.get("GITHUB_TOKEN", "")

def gh_get(path: str):
    url = f"https://api.github.com{path}"
    req = urllib.request.Request(url, headers={
        "Accept": "application/vnd.github+json",
        "Authorization": f"Bearer {TOKEN}" if TOKEN else "",
        "X-GitHub-Api-Version": "2022-11-28"
    })
    with urllib.request.urlopen(req) as r:
        return json.loads(r.read().decode())

def main():
    run = gh_get(f"/repos/{REPO}/actions/runs/{RUN_ID}")
    jobs = gh_get(f"/repos/{REPO}/actions/runs/{RUN_ID}/jobs")

    evidence = {
        "claim_id": "ARM_TTK_WORKFLOW_EXECUTION",
        "E3_execution_identity": {
            "repository": REPO,
            "workflow_name": run.get("name"),
            "run_id": run.get("id"),
            "run_attempt": run.get("run_attempt"),
            "head_sha": run.get("head_sha"),
            "head_branch": run.get("head_branch"),
            "event": run.get("event"),
            "status": run.get("status"),
            "conclusion": run.get("conclusion"),
            "created_at": run.get("created_at"),
            "updated_at": run.get("updated_at"),
            "html_url": run.get("html_url")
        },
        "E4_observed_result": {
            "jobs": [
                {"name": j.get("name"), "status": j.get("status"), "conclusion": j.get("conclusion")}
                for j in jobs.get("jobs", [])
            ]
        },
        "E7_assessment": {
            "TEST_PASSED": "NOT_ESTABLISHED",
            "note": "conclusion=action_required does not imply TEST_PASSED or TEST_FAILED. Jobs/logs required."
        },
        "E8_global": "NOT_ESTABLISHED"
    }

    os.makedirs("xa-trust/evidence", exist_ok=True)
    with open("xa-trust/evidence/run-37121498615.json", "w") as f:
        json.dump(evidence, f, indent=2)
    print(json.dumps(evidence, indent=2))

if __name__ == "__main__":
    main()


#!/usr/bin/env python3
"""XA-TRUST Verifier - E5/E6 independent recalculation"""
import json
import hashlib
import sys

def sha256_file(path: str) -> str:
    h = hashlib.sha256()
    with open(path, "rb") as f:
        h.update(f.read())
    return h.hexdigest()

def main():
    try:
        with open("xa-trust/evidence/artifact_hashes.json") as f:
            hashes = json.load(f)
        with open("xa-trust/evidence/test_execution.json") as f:
            exec_data = json.load(f)
    except FileNotFoundError as e:
        print(f"[FAIL] Missing evidence: {e}")
        sys.exit(1)

    ok = True
    for art in hashes["artifacts"]:
        actual = sha256_file(art["path"])
        if actual != art["sha256"]:
            print(f"[FAIL] Hash mismatch: {art['path']}")
            ok = False
        else:
            print(f"[OK] {art['path']} hash matches")

    failed = [r for r in exec_data["results"] if r["result"] != "PASS"]
    if failed:
        print(f"[FAIL] {len(failed)} vectors failed")
        ok = False
    else:
        print(f"[OK] All {len(exec_data['results'])} vectors PASS")

    result = {
        "E5_integrity_recalculated": "PASS" if ok else "FAIL",
        "E6_independent_verification": "PASS" if ok else "FAIL",
        "E7_claim_scoped": "VERIFIED" if ok else "NOT_VERIFIED",
        "E8_global": "NOT_ESTABLISHED"
    }
    with open("xa-trust/evidence/verification.json", "w") as f:
        json.dump(result, f, indent=2)
    print(json.dumps(result, indent=2))
    sys.exit(0 if ok else 1)

if __name__ == "__main__":
    main()

{
  "governance": "XA-TRUST / ECM",
  "GLOBAL_SCOPE": {
    "repository": "cyrillofrancisco30-lgtm/arm-ttk",
    "claims": [
      "ARM_TTK_WORKFLOW_EXECUTION",
      "ARM_TTK_TEST_RESULT",
      "ARM_TTK_ARTIFACT_INTEGRITY"
    ],
    "reference_commit": "f4555f06bcb3be2a58f033ae30336bc938431e11",
    "policy_version": "1.0.0"
  },
  "E1_claim_definition": "DEFINED",
  "E2_scope": "DEFINED",
  "E3_execution_identity": "OBSERVED - RunPester 37121498615",
  "E4_observed_result": "OBSERVED - conclusion=action_required, TEST_PASSED NOT_ESTABLISHED",
  "E5_cryptographic_binding": "REQUIRES_VERIFICATION - via verifier.py + gh attestation verify",
  "E6_independent_verification": "REQUIRES_VERIFICATION",
  "E7_claim_scoped": "NOT_PROMOTED",
  "E8_global_verified": "NOT_ESTABLISHED",
  "E9_governance_decision": "REVIEW",
  "E10_continuous_assurance": "PENDING",
  "invariants": [
    "IDENTITY != INTEGRITY != SEMANTIC_TRUTH",
    "WORKFLOW_RUN_COMPLETED ↛ TEST_PASSED",
    "WORKFLOW_CONCLUSION_SUCCESS ↛ CLAIM_CONFORMANCE",
    "TEST_PASSED ↛ VERIFIED",
    "CLAIM_SCOPED_VERIFIED ↛ GLOBAL_VERIFIED",
    "GLOBAL_VERIFIED ↛ REGULATORY_APPROVAL",
    "ELIGIBLE_FOR_PROMOTION ↛ PROMOTION_APPROVED"
  ],
  "excluded_claims": [
    "BUILD_SUCCESS_VERIFIED",
    "TEST_SUITE_PASS_VERIFIED",
    "FUNCTIONAL_INTEGRATION_VERIFIED",
    "REGULATORY_HOMOLOGATION",
    "GLOBAL_VERIFIED"
  ]
}


# XA-TRUST / ECM - arm-ttk

## E1-E10
E1 Claim -> E2 Scope -> E3 Execution Identity -> E4 Observed Result -> E5 Cryptographic Binding -> E6 Independent Verification -> E7 Claim-Scoped Verified -> E8 Global Verified -> E9 Governance Decision -> E10 Continuous Assurance

## E8 Formula
E8 = UNIVERSE_DEFINED ∧ COVERAGE_COMPLETE ∧ E7_CLAIMS_VALID ∧ CROSS_CLAIM_CONSISTENCY ∧ INDEPENDENT_RECONSTRUCTION ∧ INTEGRITY_RECALCULATED ∧ GLOBAL_POLICY_PASS

## Uso
pip install rfc8785
python3 xa-trust/producer.py
python3 xa-trust/verifier.py
GITHUB_TOKEN=xxx python3 xa-trust/collector.py

## Attestation - SEU fork, não upstream
gh attestation verify ./seu-artefato.zip --repo cyrillofrancisco30-lgtm/cli --signer-workflow cyrillofrancisco30-lgtm/cli/.github/workflows/deployment.yml --cert-oidc-issuer https://token.actions.githubusercontent.com --predicate-type https://slsa.dev/provenance/v1


pip install rfc8785
python3 xa-trust/producer.py
python3 xa-trust/verifier.py
GITHUB_TOKEN=$GITHUB_TOKEN python3 xa-trust/collector.py

# Ver evidências geradas
ls -lh xa-trust/evidence/
cat xa-trust/evidence/verification.json

# Commit e push - implementação completa
git add xa-trust/
git commit -m "feat(xa-trust): implement E1-E10 evidence producer, collector, verifier for arm-ttk"
git push origin xa-trust-implementation

# Abre PR no GitHub
gh pr create --title "feat(xa-trust): E1-E10 implementation" --body "E7 NOT_PROMOTED. E8 NOT_ESTABLISHED. Evidence in xa-trust/evidence/. Attestation must be verified on fork, not upstream cli/cli."


gh attestation verify ./seu-artefato.zip \
  --repo cyrillofrancisco30-lgtm/cli \
  --signer-workflow cyrillofrancisco30-lgtm/cli/.github/workflows/deployment.yml \
  --cert-oidc-issuer https://token.actions.githubusercontent.com \
  --predicate-type https://slsa.dev/provenance/v1 \
  --format json > xa-trust/evidence/verification-fork.json

sha256sum xa-trust/evidence/*






