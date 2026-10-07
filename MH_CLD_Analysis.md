```python
import os, re, glob, zipfile, sqlite3
import pandas as pd

# ---------- SETTINGS ----------
RAW_DIR = r"C:\Users\redbe\Desktop\ACA Expansion Project"        # folder holding your MH-CLD downloads (change if needed)
DB_PATH = "mhcld.db"
TABLE = "mhcld_records"
CHUNK = 100_000
# ------------------------------

if not os.path.isdir(RAW_DIR):
    print(f"'{RAW_DIR}' not found, searching current folder instead: {os.getcwd()}")
    RAW_DIR = "."

conn = sqlite3.connect(DB_PATH)
conn.execute(f"DROP TABLE IF EXISTS {TABLE}")   # fresh start, prevents duplicates on rerun
conn.commit()

def detect_sep(first_line):
    return "\t" if first_line.count("\t") > first_line.count(",") else ","

def load_stream(opener, label, year):
    """opener() returns a fresh binary file object for the data file."""
    with opener() as f:
        first_line = f.readline().decode("utf-8", errors="replace")
    sep = detect_sep(first_line)
    total = 0
    with opener() as f:
        for chunk in pd.read_csv(f, sep=sep, chunksize=CHUNK, low_memory=False,
                                 encoding="latin-1"):
            chunk.columns = [c.strip().upper() for c in chunk.columns]
            chunk["YEAR"] = year
            chunk.to_sql(TABLE, conn, if_exists="append", index=False)
            total += len(chunk)
    print(f"  loaded {total:,} rows from {label} (sep={'TAB' if sep=='\t' else 'COMMA'})")

DATA_EXT = (".csv", ".txt", ".tsv", ".dat")
SKIP_WORDS = ("codebook", "readme", "doc", "info")

def pick_data_member(names):
    cands = [n for n in names
             if n.lower().endswith(DATA_EXT)
             and not any(w in os.path.basename(n).lower() for w in SKIP_WORDS)]
    return cands

# Find one item per year (zip file or extracted folder)
items = glob.glob(os.path.join(RAW_DIR, "MH-CLD-*"))
found = {}
for p in items:
    m = re.search(r"MH-CLD-(\d{4})", os.path.basename(p))
    if m:
        found[int(m.group(1))] = p

print("Found:", {y: os.path.basename(p) for y, p in sorted(found.items())})

for year in [2017, 2019, 2023]:
    if year not in found:
        print(f"!! No file found for {year}. Check the name/location.")
        continue
    p = found[year]
    print(f"\n{year}: {os.path.basename(p)}")

    if zipfile.is_zipfile(p):
        zf = zipfile.ZipFile(p)
        cands = pick_data_member(zf.namelist())
        print("  data files in zip:", cands)
        if len(cands) != 1:
            print("  !! Expected exactly 1 data file. Extract the zip and check manually.")
            continue
        load_stream(lambda zf=zf, n=cands[0]: zf.open(n), cands[0], year)

    elif os.path.isdir(p):
        names = [os.path.join(dp, f) for dp, _, fs in os.walk(p) for f in fs]
        cands = pick_data_member(names)
        print("  data files in folder:", [os.path.basename(c) for c in cands])
        if len(cands) != 1:
            print("  !! Expected exactly 1 data file. Check the folder manually.")
            continue
        load_stream(lambda f=cands[0]: open(f, "rb"), os.path.basename(cands[0]), year)

    else:
        load_stream(lambda f=p: open(f, "rb"), os.path.basename(p), year)

# ---------- SANITY CHECKS ----------
print("\nRow counts by year:")
print(pd.read_sql(f"SELECT YEAR, COUNT(*) AS n FROM {TABLE} GROUP BY YEAR ORDER BY YEAR", conn))

cols = pd.read_sql(f"PRAGMA table_info({TABLE})", conn)["name"].tolist()
print(f"\n{len(cols)} columns in table:")
print(cols)

# Which columns are entirely empty for a given year? (flags variables missing/renamed across years)
print("\nColumns with zero non-null values, by year:")
for y in [2017, 2019, 2023]:
    nn = pd.read_sql(
        "SELECT " + ", ".join(f'COUNT("{c}") AS "{c}"' for c in cols if c != "YEAR")
        + f" FROM {TABLE} WHERE YEAR = {y}", conn).iloc[0]
    empty = nn[nn == 0].index.tolist()
    print(y, empty if empty else "none")
```

    Found: {2017: 'MH-CLD-2017-DS0001-bndl-data-csv_v6.zip', 2019: 'MH-CLD-2019-DS0001-bndl-data-csv_v6.zip', 2023: 'MH-CLD-2023-DS0001-bndl-data-csv_v2.zip'}
    
    2017: MH-CLD-2017-DS0001-bndl-data-csv_v6.zip
      data files in zip: ['mhcld_puf_2017.csv']
      loaded 6,172,023 rows from mhcld_puf_2017.csv (sep=COMMA)
    
    2019: MH-CLD-2019-DS0001-bndl-data-csv_v6.zip
      data files in zip: ['mhcld_puf_2019.csv']
      loaded 6,549,665 rows from mhcld_puf_2019.csv (sep=COMMA)
    
    2023: MH-CLD-2023-DS0001-bndl-data-csv_v2.zip
      data files in zip: ['mhcld_puf_2023.csv']
      loaded 7,213,557 rows from mhcld_puf_2023.csv (sep=COMMA)
    
    Row counts by year:
       YEAR        n
    0  2017  6172023
    1  2019  6549665
    2  2023  7213557
    
    40 columns in table:
    ['YEAR', 'AGE', 'EDUC', 'ETHNIC', 'RACE', 'SPHSERVICE', 'CMPSERVICE', 'OPISERVICE', 'RTCSERVICE', 'IJSSERVICE', 'MH1', 'MH2', 'MH3', 'SUB', 'MARSTAT', 'SMISED', 'SAP', 'EMPLOY', 'DETNLF', 'VETERAN', 'LIVARAG', 'NUMMHS', 'TRAUSTREFLG', 'ANXIETYFLG', 'ADHDFLG', 'CONDUCTFLG', 'DELIRDEMFLG', 'BIPOLARFLG', 'DEPRESSFLG', 'ODDFLG', 'PDDFLG', 'PERSONFLG', 'SCHIZOFLG', 'ALCSUBFLG', 'OTHERDISFLG', 'STATEFIP', 'DIVISION', 'REGION', 'CASEID', 'SEX']
    
    Columns with zero non-null values, by year:
    2017 none
    2019 none
    2023 none
    


```python
import sqlite3
import pandas as pd
conn = sqlite3.connect("mhcld.db")

checks = ["NUMMHS", "EMPLOY", "LIVARAG", "SMISED", "SAP", "SEX", "DIVISION", "SPHSERVICE"]

for col in checks:
    print(f"\n=== {col} ===")
    df = pd.read_sql(
        f"SELECT YEAR, {col} AS value, COUNT(*) AS n "
        f"FROM mhcld_records GROUP BY YEAR, {col} ORDER BY {col}, YEAR", conn)
    print(df.pivot(index="value", columns="YEAR", values="n"))

print("\nDistinct states per year:")
print(pd.read_sql(
    "SELECT YEAR, COUNT(DISTINCT STATEFIP) AS n_states "
    "FROM mhcld_records GROUP BY YEAR", conn))

print("\nStates in 2017 but not 2019 or 2023, and vice versa:")
states = pd.read_sql(
    "SELECT DISTINCT YEAR, STATEFIP FROM mhcld_records", conn)
by_year = {y: set(g["STATEFIP"]) for y, g in states.groupby("YEAR")}
print({y: sorted(by_year[y] - by_year[2017]) for y in (2019, 2023)}, "(new vs 2017)")
print({y: sorted(by_year[2017] - by_year[y]) for y in (2019, 2023)}, "(dropped vs 2017)")
```

    
    === NUMMHS ===
    YEAR      2017     2019     2023
    value                           
    0       922671   800867   946853
    1      3519858  3708647  3809717
    2      1374661  1572686  1743879
    3       354833   467465   713108
    
    === EMPLOY ===
    YEAR      2017     2019     2023
    value                           
    -9     3861035  4194896  4382724
     1      240669   270105   443407
     2      166132   174899   213333
     3       70555   100944   130948
     4      676135   667375   778478
     5     1157497  1141446  1264667
    
    === LIVARAG ===
    YEAR      2017     2019     2023
    value                           
    -9     2509741  2385700  2637659
     1      141126   172772   200415
     2     3061047  3561893  3978372
     3      460109   429300   397111
    
    === SMISED ===
    YEAR      2017     2019     2023
    value                           
    -9      342811   312035   471630
     1     2983739  3349594  3617089
     2     1147771  1424823  1319844
     3     1697702  1463213  1804994
    
    === SAP ===
    YEAR      2017     2019     2023
    value                           
    -9      821602   789385   617646
     1     1188127  1574529  2561712
     2     4162294  4185751  4034199
    
    === SEX ===
    YEAR      2017     2019     2023
    value                           
    -9       26855    17602    13626
     1     2991603  3168724  3400552
     2     3153565  3363339  3799379
    
    === DIVISION ===
    YEAR      2017     2019     2023
    value                           
    0        25918     4473     4316
    1       240464   183285   217578
    2      1033948  1071901   985360
    3       990665  1076064   627941
    4       501542   689183   963008
    5       812765   821994  1116697
    6       441287   469034   449584
    7       600611   635386   691675
    8       650495   521422   960324
    9       874328  1076923  1197074
    
    === SPHSERVICE ===
    YEAR      2017     2019     2023
    value                           
    1       124273   125260   113723
    2      6047750  6424405  7099834
    
    Distinct states per year:
       YEAR  n_states
    0  2017        48
    1  2019        47
    2  2023        52
    
    States in 2017 but not 2019 or 2023, and vice versa:
    {2019: [2], 2023: [2, 13, 20, 33, 54]} (new vs 2017)
    {2019: [4, 23], 2023: [23]} (dropped vs 2017)
    


```python
import sqlite3
import pandas as pd
conn = sqlite3.connect("mhcld.db")

# State FIPS -> year expansion took effect (None = never, as of 2023)
expansion_year = {
    # 2014
    4:2014, 5:2014, 6:2014, 8:2014, 9:2014, 10:2014, 11:2014, 15:2014, 17:2014,
    19:2014, 21:2014, 24:2014, 25:2014, 26:2014, 27:2014, 32:2014, 33:2014,
    34:2014, 35:2014, 36:2014, 38:2014, 39:2014, 41:2014, 44:2014, 50:2014,
    53:2014, 54:2014,
    # 2015-2016
    2:2015, 18:2015, 42:2015, 22:2016, 30:2016,
    # 2019-2021
    23:2019, 51:2019, 16:2020, 31:2020, 49:2020, 29:2021, 40:2021,
    # Mid/late 2023: coded as 2024 so they are NOT treated in the 2023 data
    37:2024, 46:2024,
}
never = [1, 12, 13, 20, 28, 45, 47, 48, 55, 56]   # AL FL GA KS MS SC TN TX WI WY

rows = list(expansion_year.items()) + [(f, None) for f in never]
exp_df = pd.DataFrame(rows, columns=["STATEFIP", "EXPANSION_YEAR"])
exp_df.to_sql("expansion_dates", conn, if_exists="replace", index=False)
print(len(exp_df), "jurisdictions in expansion table")

# Which STATEFIP codes in the data have no match in the table?
data_states = pd.read_sql("SELECT DISTINCT YEAR, STATEFIP FROM mhcld_records", conn)
unmatched = data_states[~data_states["STATEFIP"].isin(exp_df["STATEFIP"])]
print("\nCodes in data but not in expansion table:")
print(unmatched.sort_values(["STATEFIP", "YEAR"]).to_string(index=False))

# Value counts for columns not yet inspected
for col in ["CMPSERVICE", "OPISERVICE", "RTCSERVICE", "IJSSERVICE",
            "AGE", "EDUC", "RACE", "ETHNIC"]:
    print(f"\n=== {col} ===")
    df = pd.read_sql(
        f"SELECT YEAR, {col} AS value, COUNT(*) AS n "
        f"FROM mhcld_records GROUP BY YEAR, {col} ORDER BY {col}, YEAR", conn)
    print(df.pivot(index="value", columns="YEAR", values="n"))
```

    51 jurisdictions in expansion table
    
    Codes in data but not in expansion table:
     YEAR  STATEFIP
     2017        72
     2019        72
     2023        72
     2017        99
     2019        99
     2023        99
    
    === CMPSERVICE ===
    YEAR      2017     2019     2023
    value                           
    1      5994358  6379727  6987818
    2       177665   169938   225739
    
    === OPISERVICE ===
    YEAR      2017     2019     2023
    value                           
    1       264246   265133   279882
    2      5907777  6284532  6933675
    
    === RTCSERVICE ===
    YEAR      2017     2019     2023
    value                           
    1        67657    75665    67848
    2      6104366  6474000  7145709
    
    === IJSSERVICE ===
    YEAR      2017     2019     2023
    value                           
    1        72568    72731    77113
    2      6099455  6476934  7136444
    
    === AGE ===
    YEAR     2017    2019    2023
    value                        
    -9       9266    3121    5321
     1     814582  919934  874537
     2     405354  478986  526981
     3     450173  494242  557952
     4     273838  310240  364194
     5     353443  366947  432073
     6     536938  569053  614447
     7     521614  548210  674162
     8     481177  509814  589757
     9     405028  416749  517028
     10    434899  419051  411839
     11    466051  434873  418397
     12    418444  429084  413347
     13    278848  307438  363884
     14    322368  341923  449638
    
    === EDUC ===
    YEAR      2017     2019     2023
    value                           
    -9     3475283  3567993  3624639
     1       27662    34273    35506
     2      703066   795855   934129
     3      524751   549551   641996
     4      962665  1068423  1327812
     5      478596   533570   649475
    
    === RACE ===
    YEAR      2017     2019     2023
    value                           
    -9      535013   563002   890575
     1       79046   102524   154840
     2       85859    90949   110864
     3     1159255  1240623  1234202
     4       10968    15511    24855
     5     3742010  3893214  4058529
     6      559872   643842   739692
    
    === ETHNIC ===
    YEAR      2017     2019     2023
    value                           
    -9      632518   767182   997686
     1       52812    54382    52806
     2       51221    27017    36462
     3      795856   898500  1306425
     4     4639616  4802584  4820178
    


```python
import sqlite3
import pandas as pd
conn = sqlite3.connect("mhcld.db")

conn.execute("CREATE INDEX IF NOT EXISTS idx_year_state ON mhcld_records(YEAR, STATEFIP)")
conn.execute("DROP VIEW IF EXISTS clean_records")
conn.execute("""
CREATE VIEW clean_records AS
SELECT
    r.CASEID, r.YEAR, r.STATEFIP, r.DIVISION,
    r.NUMMHS                                   AS nummhs,
    CASE WHEN r.NUMMHS >= 1 THEN 1 ELSE 0 END  AS any_dx,
    CASE WHEN r.NUMMHS >= 2 THEN 1 ELSE 0 END  AS multi_dx,
    CASE WHEN e.EXPANSION_YEAR IS NOT NULL
          AND e.EXPANSION_YEAR <= r.YEAR THEN 1 ELSE 0 END AS expansion,

    CASE WHEN r.EMPLOY = -9 THEN NULL WHEN r.EMPLOY IN (1,2,3) THEN 1 ELSE 0 END AS employed,
    CASE WHEN r.EMPLOY = -9 THEN NULL WHEN r.EMPLOY = 4 THEN 1 ELSE 0 END        AS unemployed,
    CASE WHEN r.EMPLOY = -9 THEN NULL WHEN r.EMPLOY = 5 THEN 1 ELSE 0 END        AS not_in_labor_force,
    CASE WHEN r.LIVARAG = -9 THEN NULL WHEN r.LIVARAG = 1 THEN 1 ELSE 0 END      AS homeless,
    CASE WHEN r.SMISED = -9 THEN NULL WHEN r.SMISED IN (1,2) THEN 1 ELSE 0 END   AS smi_sed,
    CASE WHEN r.SAP = -9 THEN NULL WHEN r.SAP = 1 THEN 1 ELSE 0 END              AS sud,

    CASE WHEN r.SPHSERVICE = 1 THEN 1 ELSE 0 END AS svc_state_hosp,
    CASE WHEN r.CMPSERVICE = 1 THEN 1 ELSE 0 END AS svc_community,
    CASE WHEN r.OPISERVICE = 1 THEN 1 ELSE 0 END AS svc_other_inpatient,  -- omitted category in models
    CASE WHEN r.RTCSERVICE = 1 THEN 1 ELSE 0 END AS svc_residential,
    CASE WHEN r.IJSSERVICE = 1 THEN 1 ELSE 0 END AS svc_justice,

    CASE WHEN r.SEX = -9 THEN NULL WHEN r.SEX = 2 THEN 1 ELSE 0 END AS female,
    NULLIF(r.AGE, -9)    AS age_grp,
    CASE WHEN r.AGE = -9 THEN NULL WHEN r.AGE >= 4 THEN 1 ELSE 0 END AS adult,
    NULLIF(r.EDUC, -9)   AS educ,
    NULLIF(r.RACE, -9)   AS race,
    NULLIF(r.ETHNIC, -9) AS ethnic
FROM mhcld_records r
LEFT JOIN expansion_dates e ON e.STATEFIP = r.STATEFIP
WHERE r.STATEFIP NOT IN (72, 99)
  AND r.DIVISION BETWEEN 1 AND 9
""")
conn.commit()
print("view created")
```

    view created
    


```python
chk = pd.read_sql("""
SELECT COUNT(*) AS n,
       AVG(nummhs) AS nummhs, AVG(expansion) AS expansion, AVG(employed) AS employed,
       AVG(homeless) AS homeless, AVG(svc_state_hosp) AS state_hosp,
       AVG(svc_community) AS community, AVG(svc_other_inpatient) AS other_inpt,
       AVG(svc_residential) AS residential, AVG(svc_justice) AS justice,
       AVG(smi_sed) AS smi_sed, AVG(sud) AS sud, AVG(female) AS female
FROM clean_records WHERE YEAR = 2017
""", conn).T
chk["thesis_table2"] = [6146072, 1.193, .732, .208, .0385, .0198, .972, .0427, .011, .012, .708, .219, .514]
print(chk)
```

                            0  thesis_table2
    n            6.146105e+06   6.146072e+06
    nummhs       1.192782e+00   1.193000e+00
    expansion    7.323077e-01   7.320000e-01
    employed     2.072376e-01   2.080000e-01
    homeless     3.865689e-02   3.850000e-02
    state_hosp   1.976536e-02   1.980000e-02
    community    9.718109e-01   9.720000e-01
    other_inpt   4.269484e-02   4.270000e-02
    residential  1.099200e-02   1.100000e-02
    justice      1.180699e-02   1.200000e-02
    smi_sed      7.093857e-01   7.080000e-01
    sud          2.210625e-01   2.190000e-01
    female       5.133611e-01   5.140000e-01
    


```python
attr = pd.read_sql("""
SELECT YEAR,
  COUNT(*)                                                         AS all_valid_state,
  SUM(adult = 1)                                                   AS adults,
  SUM(adult = 1 AND homeless IS NOT NULL AND smi_sed IS NOT NULL
      AND sud IS NOT NULL AND female IS NOT NULL AND race IS NOT NULL
      AND ethnic IS NOT NULL AND age_grp IS NOT NULL)              AS adults_core_controls,
  SUM(adult = 1 AND homeless IS NOT NULL AND smi_sed IS NOT NULL
      AND sud IS NOT NULL AND female IS NOT NULL AND race IS NOT NULL
      AND ethnic IS NOT NULL AND age_grp IS NOT NULL
      AND employed IS NOT NULL AND educ IS NOT NULL)               AS adults_all_controls
FROM clean_records GROUP BY YEAR ORDER BY YEAR
""", conn)
print(attr)
```

       YEAR  all_valid_state   adults  adults_core_controls  adults_all_controls
    0  2017          6146105  4469740               2005100              1177086
    1  2019          6545192  4649718               2312158              1437494
    2  2023          7209241  5245609               2647977              1692400
    


```python
!pip install statsmodels
import sqlite3
import pandas as pd
import statsmodels.formula.api as smf

conn = sqlite3.connect("mhcld.db")

df = pd.read_sql("""
SELECT nummhs, expansion, employed, homeless, svc_state_hosp, svc_community,
       svc_residential, svc_justice, smi_sed, sud, DIVISION, STATEFIP,
       age_grp, educ, female, race, ethnic
FROM clean_records
WHERE YEAR = 2017
  AND employed IS NOT NULL AND homeless IS NOT NULL AND smi_sed IS NOT NULL
  AND sud IS NOT NULL AND age_grp IS NOT NULL AND educ IS NOT NULL
  AND female IS NOT NULL AND race IS NOT NULL AND ethnic IS NOT NULL
""", conn)
print(f"Regression 3 sample: {len(df):,} rows (thesis: 1,223,938)")

# Dummies matching the thesis (omitted: other/mixed race, non-Mexican/non-Puerto Rican ethnicity)
df["native"]       = (df["race"] == 1).astype(int)
df["asian"]        = (df["race"] == 2).astype(int)
df["black"]        = (df["race"] == 3).astype(int)
df["pacific"]      = (df["race"] == 4).astype(int)
df["white"]        = (df["race"] == 5).astype(int)
df["mexican"]      = (df["ethnic"] == 1).astype(int)
df["puerto_rican"] = (df["ethnic"] == 2).astype(int)

formula = ("nummhs ~ expansion + employed + homeless + svc_state_hosp + svc_community"
           " + svc_residential + svc_justice + smi_sed + sud"
           " + C(DIVISION, Treatment(reference=9))"
           " + age_grp + educ + female + native + asian + black + pacific + white"
           " + mexican + puerto_rican")

model = smf.ols(formula, data=df)
res_hc   = model.fit(cov_type="HC1")
res_clus = model.fit(cov_type="cluster", cov_kwds={"groups": df["STATEFIP"]})

out = pd.DataFrame({
    "coef": res_hc.params,
    "se_robust": res_hc.bse,
    "se_state_clustered": res_clus.bse,
})
print(out.round(4))
print("\nNumber of state clusters:", df["STATEFIP"].nunique())
print("Expansion coef:", round(res_hc.params["expansion"], 4), "(thesis: .254)")
```

    Requirement already satisfied: statsmodels in C:\Users\redbe\Coding_Anaconda\Lib\site-packages (0.14.6)
    Requirement already satisfied: numpy<3,>=1.22.3 in C:\Users\redbe\Coding_Anaconda\Lib\site-packages (from statsmodels) (2.4.6)
    Requirement already satisfied: scipy!=1.9.2,>=1.8 in C:\Users\redbe\Coding_Anaconda\Lib\site-packages (from statsmodels) (1.18.0)
    Requirement already satisfied: pandas!=2.1.0,>=1.4 in C:\Users\redbe\Coding_Anaconda\Lib\site-packages (from statsmodels) (3.0.3)
    Requirement already satisfied: patsy>=0.5.6 in C:\Users\redbe\Coding_Anaconda\Lib\site-packages (from statsmodels) (1.0.2)
    Requirement already satisfied: packaging>=21.3 in C:\Users\redbe\Coding_Anaconda\Lib\site-packages (from statsmodels) (26.0)
    Requirement already satisfied: python-dateutil>=2.8.2 in C:\Users\redbe\Coding_Anaconda\Lib\site-packages (from pandas!=2.1.0,>=1.4->statsmodels) (2.9.0.post0)
    Requirement already satisfied: tzdata in C:\Users\redbe\Coding_Anaconda\Lib\site-packages (from pandas!=2.1.0,>=1.4->statsmodels) (2026.2)
    Requirement already satisfied: six>=1.5 in C:\Users\redbe\Coding_Anaconda\Lib\site-packages (from python-dateutil>=2.8.2->pandas!=2.1.0,>=1.4->statsmodels) (1.17.0)
    Regression 3 sample: 1,227,549 rows (thesis: 1,223,938)
                                                coef  se_robust  \
    Intercept                                 0.6519     0.0088   
    C(DIVISION, Treatment(reference=9))[T.1] -0.1891     0.0051   
    C(DIVISION, Treatment(reference=9))[T.2]  0.0700     0.0049   
    C(DIVISION, Treatment(reference=9))[T.3]  0.2368     0.0034   
    C(DIVISION, Treatment(reference=9))[T.4]  0.3612     0.0036   
    C(DIVISION, Treatment(reference=9))[T.5]  0.4421     0.0044   
    C(DIVISION, Treatment(reference=9))[T.6]  0.3105     0.0037   
    C(DIVISION, Treatment(reference=9))[T.7]  0.2825     0.0040   
    C(DIVISION, Treatment(reference=9))[T.8]  0.2313     0.0038   
    expansion                                 0.2533     0.0022   
    employed                                 -0.0203     0.0016   
    homeless                                 -0.0123     0.0034   
    svc_state_hosp                            0.0420     0.0051   
    svc_community                             0.2273     0.0067   
    svc_residential                           0.0449     0.0071   
    svc_justice                               0.0109     0.0129   
    smi_sed                                   0.2644     0.0017   
    sud                                       0.1495     0.0016   
    age_grp                                  -0.0119     0.0002   
    educ                                      0.0060     0.0007   
    female                                    0.1113     0.0013   
    native                                   -0.0618     0.0057   
    asian                                    -0.1088     0.0063   
    black                                    -0.1253     0.0032   
    pacific                                  -0.0695     0.0155   
    white                                    -0.0144     0.0029   
    mexican                                  -0.1420     0.0053   
    puerto_rican                              0.0642     0.0086   
    
                                              se_state_clustered  
    Intercept                                             0.1993  
    C(DIVISION, Treatment(reference=9))[T.1]              0.2469  
    C(DIVISION, Treatment(reference=9))[T.2]              0.0471  
    C(DIVISION, Treatment(reference=9))[T.3]              0.0614  
    C(DIVISION, Treatment(reference=9))[T.4]              0.0950  
    C(DIVISION, Treatment(reference=9))[T.5]              0.2447  
    C(DIVISION, Treatment(reference=9))[T.6]              0.0950  
    C(DIVISION, Treatment(reference=9))[T.7]              0.1557  
    C(DIVISION, Treatment(reference=9))[T.8]              0.0950  
    expansion                                             0.1261  
    employed                                              0.0167  
    homeless                                              0.0273  
    svc_state_hosp                                        0.1016  
    svc_community                                         0.0887  
    svc_residential                                       0.1019  
    svc_justice                                           0.0818  
    smi_sed                                               0.0728  
    sud                                                   0.0486  
    age_grp                                               0.0037  
    educ                                                  0.0066  
    female                                                0.0179  
    native                                                0.0472  
    asian                                                 0.0224  
    black                                                 0.0226  
    pacific                                               0.0414  
    white                                                 0.0233  
    mexican                                               0.0434  
    puerto_rican                                          0.0886  
    
    Number of state clusters: 36
    Expansion coef: 0.2533 (thesis: .254)
    


```python
import pandas as pd
import sqlite3
conn = sqlite3.connect("mhcld.db")

minimal = ("adult = 1 AND smi_sed IS NOT NULL AND sud IS NOT NULL AND female IS NOT NULL "
           "AND race IS NOT NULL AND ethnic IS NOT NULL AND age_grp IS NOT NULL")
core = minimal + " AND homeless IS NOT NULL"
full = core + " AND employed IS NOT NULL AND educ IS NOT NULL"
tiers = {"adults": "adult = 1", "minimal": minimal, "core": core, "full": full}

parts = []
for name, cond in tiers.items():
    parts.append(f"""
    SELECT YEAR, '{name}' AS sample_tier, COUNT(*) AS n,
           COUNT(DISTINCT STATEFIP) AS n_states,
           ROUND(AVG(expansion), 3) AS share_expansion,
           ROUND(AVG(nummhs), 3) AS mean_nummhs
    FROM clean_records WHERE {cond} GROUP BY YEAR""")
print(pd.read_sql(" UNION ALL ".join(parts) + " ORDER BY YEAR, n DESC", conn))

# Is missingness a state-level reporting issue? (2017, adults)
miss = pd.read_sql("""
SELECT STATEFIP, COUNT(*) AS n,
       ROUND(AVG(homeless IS NULL), 2) AS pct_missing_homeless,
       ROUND(AVG(employed IS NULL), 2) AS pct_missing_employ,
       ROUND(AVG(educ IS NULL), 2)     AS pct_missing_educ
FROM clean_records WHERE YEAR = 2017 AND adult = 1
GROUP BY STATEFIP ORDER BY pct_missing_homeless DESC
""", conn)
print(miss.to_string(index=False))
```

        YEAR sample_tier        n  n_states  share_expansion  mean_nummhs
    0   2017      adults  4469740        46            0.713        1.190
    1   2017     minimal  3211610        42            0.792        1.282
    2   2017        core  2005100        42            0.721        1.404
    3   2017        full  1177086        36            0.613        1.484
    4   2019      adults  4649718        45            0.751        1.271
    5   2019     minimal  3489695        43            0.779        1.338
    6   2019        core  2312158        43            0.729        1.433
    7   2019        full  1437494        39            0.634        1.523
    8   2023      adults  5245609        50            0.765        1.336
    9   2023     minimal  3783907        48            0.769        1.428
    10  2023        core  2647977        48            0.713        1.494
    11  2023        full  1692400        40            0.644        1.581
     STATEFIP      n  pct_missing_homeless  pct_missing_employ  pct_missing_educ
           23  49494                  1.00                0.11              1.00
           11  31159                  0.99                1.00              1.00
           42 393344                  0.99                0.90              1.00
           39 369601                  0.97                0.98              0.99
           35 101824                  0.92                0.96              0.81
            4 121742                  0.87                0.76              1.00
           19  82009                  0.85                0.88              1.00
           15   7683                  0.71                0.74              0.54
           46  10694                  0.68                0.69              0.93
           16  13546                  0.65                0.01              0.99
           55  54042                  0.64                0.67              0.98
           22  41719                  0.61                0.67              0.50
           30  26831                  0.54                0.71              0.76
            5  61851                  0.46                0.19              1.00
           50  20979                  0.45                0.60              1.00
            6 411583                  0.39                0.79              1.00
           47  92624                  0.34                0.45              0.68
           32  15994                  0.32                0.58              0.99
            1  71269                  0.31                0.29              0.03
           48 296308                  0.31                0.32              0.31
           44  23945                  0.30                0.32              1.00
            9  61744                  0.28                0.32              0.39
           53  68681                  0.27                0.35              0.32
           10   7028                  0.24                0.35              0.32
           31  19524                  0.21                0.11              0.27
           25  26964                  0.19                0.27              0.46
           40  73029                  0.18                0.20              0.23
           12 176348                  0.17                0.16              0.15
           36  50418                  0.17                0.46              0.11
           29  60188                  0.16                0.30              0.41
           34 345363                  0.16                0.91              0.18
           37 127524                  0.16                0.04              0.10
           41  93226                  0.16                0.52              0.52
           49  33414                  0.08                0.12              0.14
           45  57788                  0.07                0.93              0.23
           38  11011                  0.06                0.14              1.00
           51  86997                  0.06                0.07              0.10
           17  52843                  0.04                0.14              0.14
           21 103992                  0.02                0.07              0.05
           28  46893                  0.02                0.09              0.05
           56  12563                  0.02                0.05              0.01
           18  82970                  0.01                0.02              0.03
           24 135015                  0.01                0.52              0.99
            8 103062                  0.00                0.01              0.06
           26 140865                  0.00                0.01              0.08
           27 194049                  0.00                0.01              0.04
    


```python
import sqlite3
import pandas as pd
import statsmodels.formula.api as smf

conn = sqlite3.connect("mhcld.db")

formula = ("nummhs ~ expansion + employed + homeless + svc_state_hosp + svc_community"
           " + svc_residential + svc_justice + smi_sed + sud"
           " + C(DIVISION, Treatment(reference=9))"
           " + age_grp + educ + female + native + asian + black + pacific + white"
           " + mexican + puerto_rican")

def run_year(year):
    df = pd.read_sql(f"""
    SELECT nummhs, expansion, employed, homeless, svc_state_hosp, svc_community,
           svc_residential, svc_justice, smi_sed, sud, DIVISION, STATEFIP,
           age_grp, educ, female, race, ethnic
    FROM clean_records
    WHERE YEAR = {year}
      AND employed IS NOT NULL AND homeless IS NOT NULL AND smi_sed IS NOT NULL
      AND sud IS NOT NULL AND age_grp IS NOT NULL AND educ IS NOT NULL
      AND female IS NOT NULL AND race IS NOT NULL AND ethnic IS NOT NULL
    """, conn)
    for name, code in [("native",1),("asian",2),("black",3),("pacific",4),("white",5)]:
        df[name] = (df["race"] == code).astype(int)
    df["mexican"] = (df["ethnic"] == 1).astype(int)
    df["puerto_rican"] = (df["ethnic"] == 2).astype(int)

    model = smf.ols(formula, data=df)
    r_hc = model.fit(cov_type="HC1")
    r_cl = model.fit(cov_type="cluster", cov_kwds={"groups": df["STATEFIP"]})
    b = r_cl.params["expansion"]
    ci = r_cl.conf_int().loc["expansion"]
    return {"year": year, "n": len(df), "states": df["STATEFIP"].nunique(),
            "share_expansion": round(df["expansion"].mean(), 3),
            "coef": round(b, 4), "se_robust": round(r_hc.bse["expansion"], 4),
            "se_clustered": round(r_cl.bse["expansion"], 4),
            "ci_low": round(ci[0], 3), "ci_high": round(ci[1], 3)}

results = pd.DataFrame([run_year(y) for y in (2017, 2019, 2023)])
print(results.to_string(index=False))
```

     year       n  states  share_expansion    coef  se_robust  se_clustered  ci_low  ci_high
     2017 1227549      36            0.614  0.2533     0.0022        0.1261   0.006    0.500
     2019 1517092      39            0.640  0.3765     0.0017        0.1721   0.039    0.714
     2023 1786855      40            0.651 -0.0002     0.0017        0.2389  -0.468    0.468
    


```python
import sqlite3
import pandas as pd
import statsmodels.formula.api as smf

conn = sqlite3.connect("mhcld.db")

MINIMAL = ("adult = 1 AND smi_sed IS NOT NULL AND sud IS NOT NULL AND female IS NOT NULL "
           "AND race IS NOT NULL AND ethnic IS NOT NULL AND age_grp IS NOT NULL")
FULL = MINIMAL + " AND homeless IS NOT NULL AND employed IS NOT NULL AND educ IS NOT NULL"

def load(year, cond):
    df = pd.read_sql(f"""
        SELECT nummhs, expansion, DIVISION, STATEFIP, smi_sed, sud, female, age_grp, race, ethnic
        FROM clean_records WHERE YEAR = {year} AND {cond}""", conn)
    for c in ["nummhs", "expansion", "smi_sed", "sud", "female", "age_grp"]:
        df[c] = df[c].astype("float32")
    return df

specs = {
    "A: expansion only":          "nummhs ~ expansion",
    "B: + division":              "nummhs ~ expansion + C(DIVISION)",
    "C: + minimal controls":      "nummhs ~ expansion + C(DIVISION) + smi_sed + sud + female + age_grp + C(race) + C(ethnic)",
}

rows = []
for year in (2017, 2019, 2023):
    dfm = load(year, MINIMAL)
    for name, f in specs.items():
        r = smf.ols(f, data=dfm).fit(cov_type="cluster", cov_kwds={"groups": dfm["STATEFIP"]})
        rows.append({"year": year, "sample": "minimal", "spec": name, "n": len(dfm),
                     "states": dfm["STATEFIP"].nunique(),
                     "coef": round(r.params["expansion"], 4), "se": round(r.bse["expansion"], 4)})
    del dfm
    dff = load(year, FULL)
    r = smf.ols("nummhs ~ expansion", data=dff).fit(cov_type="cluster", cov_kwds={"groups": dff["STATEFIP"]})
    rows.append({"year": year, "sample": "full", "spec": "A: expansion only", "n": len(dff),
                 "states": dff["STATEFIP"].nunique(),
                 "coef": round(r.params["expansion"], 4), "se": round(r.bse["expansion"], 4)})
    del dff

print(pd.DataFrame(rows).to_string(index=False))
```

     year  sample                  spec       n  states    coef     se
     2017 minimal     A: expansion only 3211610      42 -0.1039 0.0957
     2017 minimal         B: + division 3211610      42 -0.0048 0.1117
     2017 minimal C: + minimal controls 3211610      42  0.0971 0.1074
     2017    full     A: expansion only 1177086      36  0.1082 0.0930
     2019 minimal     A: expansion only 3489695      43  0.0811 0.1424
     2019 minimal         B: + division 3489695      43  0.2230 0.1469
     2019 minimal C: + minimal controls 3489695      43  0.3047 0.1458
     2019    full     A: expansion only 1437494      39  0.2676 0.1591
     2023 minimal     A: expansion only 3783907      48  0.1017 0.2001
     2023 minimal         B: + division 3783907      48  0.0578 0.1907
     2023 minimal C: + minimal controls 3783907      48  0.0832 0.1895
     2023    full     A: expansion only 1692400      40  0.2521 0.2465
    


```python
panel = pd.read_sql(f"""
    SELECT YEAR, STATEFIP, MAX(expansion) AS expansion, COUNT(*) AS n,
           AVG(nummhs) AS nummhs, AVG(smi_sed) AS smi_sed, AVG(sud) AS sud,
           AVG(female) AS female, AVG(age_grp) AS age_grp
    FROM clean_records
    WHERE YEAR IN (2017, 2019, 2023) AND {MINIMAL}
    GROUP BY YEAR, STATEFIP""", conn)

print(len(panel), "state-year cells")

# Which states change expansion status between observed years?
status = panel.groupby("STATEFIP")["expansion"].agg(["min", "max", "count"])
switchers = status[(status["min"] != status["max"])]
print("\nStates that switch status within the data:", list(switchers.index))

f = "nummhs ~ expansion + smi_sed + sud + female + age_grp + C(STATEFIP) + C(YEAR)"
fe = smf.wls(f, data=panel, weights=panel["n"]).fit(
    cov_type="cluster", cov_kwds={"groups": panel["STATEFIP"]})
ci = fe.conf_int().loc["expansion"]
print(f"\nState+year FE: coef={fe.params['expansion']:.4f}  se={fe.bse['expansion']:.4f}  "
      f"CI=({ci[0]:.3f}, {ci[1]:.3f})")
```

    133 state-year cells
    
    States that switch status within the data: [16, 29, 31, 40, 49, 51]
    
    State+year FE: coef=0.0513  se=0.1199  CI=(-0.184, 0.286)
    


```python
import sqlite3
import pandas as pd
import statsmodels.formula.api as smf

conn = sqlite3.connect("mhcld.db")

MINIMAL = ("adult = 1 AND smi_sed IS NOT NULL AND sud IS NOT NULL AND female IS NOT NULL "
           "AND race IS NOT NULL AND ethnic IS NOT NULL AND age_grp IS NOT NULL")
FULL = MINIMAL + " AND homeless IS NOT NULL AND employed IS NOT NULL AND educ IS NOT NULL"

def load(year, cond):
    df = pd.read_sql(f"""
        SELECT nummhs, expansion, DIVISION, STATEFIP, smi_sed, sud, female, age_grp,
               race, ethnic, svc_state_hosp, svc_community, svc_residential, svc_justice,
               homeless, employed, educ
        FROM clean_records WHERE YEAR = {year} AND {cond}""", conn)
    num = ["nummhs","expansion","smi_sed","sud","female","age_grp","svc_state_hosp",
           "svc_community","svc_residential","svc_justice","homeless","employed","educ"]
    df[num] = df[num].astype("float32")
    return df

base  = "nummhs ~ expansion + C(DIVISION) + smi_sed + sud + female + age_grp + C(race) + C(ethnic)"
svc   = " + svc_state_hosp + svc_community + svc_residential + svc_justice"
extra = " + homeless + employed + educ"
specs = {
    "A_expansion_only":   ("nummhs ~ expansion", ("minimal", "full")),
    "C_division+demog":   (base, ("minimal", "full")),
    "D_+service_type":    (base + svc, ("minimal", "full")),
    "E_+homeless/emp/ed": (base + svc + extra, ("full",)),
}

rows = []
for year in (2017, 2019, 2023):
    for sample, cond in (("minimal", MINIMAL), ("full", FULL)):
        df = load(year, cond)
        for name, (f, allowed) in specs.items():
            if sample not in allowed:
                continue
            r = smf.ols(f, data=df).fit(cov_type="cluster", cov_kwds={"groups": df["STATEFIP"]})
            rows.append({"year": year, "sample": sample, "spec": name,
                         "coef": round(r.params["expansion"], 3),
                         "se": round(r.bse["expansion"], 3),
                         "states": df["STATEFIP"].nunique()})
        del df

res = pd.DataFrame(rows)
print(res.pivot_table(index=["year", "sample"], columns="spec", values="coef", sort=False))
print()
print(res.pivot_table(index=["year", "sample"], columns="spec", values="se", sort=False))
```

    spec          A_expansion_only  C_division+demog  D_+service_type  \
    year sample                                                         
    2017 minimal            -0.104             0.097            0.099   
         full                0.108             0.284            0.287   
    2019 minimal             0.081             0.305            0.304   
         full                0.268             0.382            0.380   
    2023 minimal             0.102             0.083            0.079   
         full                0.252             0.005           -0.004   
    
    spec          E_+homeless/emp/ed  
    year sample                       
    2017 minimal                 NaN  
         full                  0.289  
    2019 minimal                 NaN  
         full                  0.380  
    2023 minimal                 NaN  
         full                 -0.004  
    
    spec          A_expansion_only  C_division+demog  D_+service_type  \
    year sample                                                         
    2017 minimal             0.096             0.107            0.107   
         full                0.093             0.125            0.124   
    2019 minimal             0.142             0.146            0.145   
         full                0.159             0.175            0.174   
    2023 minimal             0.200             0.190            0.188   
         full                0.247             0.243            0.242   
    
    spec          E_+homeless/emp/ed  
    year sample                       
    2017 minimal                 NaN  
         full                  0.125  
    2019 minimal                 NaN  
         full                  0.174  
    2023 minimal                 NaN  
         full                  0.241  
    


```python

```
