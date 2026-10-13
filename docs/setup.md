
# 需要 Java 17（Synthea 和 PySpark 都要）
sudo apt update && sudo apt install -y openjdk-17-jdk
java -version

# 建專案
mkdir -p ~/projects/fhir-lakehouse && cd ~/projects/fhir-lakehouse
git init
uv init
uv add pyspark delta-spark pytest ruff   # 兩者版本要相容，依 Delta 官方的相容表確認
mkdir -p src/ingest src/transform src/quality sql tests/fixtures docs tools

# 資料不進 Git
printf "data/\noutput/\ntools/*.jar\n.venv/\n" > .gitignore

# 下載 Synthea（synthea-with-dependencies.jar，從 Synthea 的 GitHub Releases）放進 tools/，再產生資料
mkdir -p tools
curl -L -o tools/synthea-with-dependencies.jar \
  https://github.com/synthetichealth/synthea/releases/download/master-branch-latest/synthea-with-dependencies.jar
ls -lh tools/   # 確認檔案存在，大小應該有一兩百 MB
java -jar tools/synthea-with-dependencies.jar -p 1000 -s 42 --exporter.fhir.bulk_data true

wc -l output/fhir/Patient.ndjson                       # 總人數
grep -c '"deceasedDateTime"' output/fhir/Patient.ndjson  # 已故人數