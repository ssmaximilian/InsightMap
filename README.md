# InsightMap
The results will be the quantitative values along with maps depicting relevant geographies of consumer complaints.

├── README.md
├── run.sh
├── src
│   └── consumer_complaints.py
├── input
│   └── complaints.csv
├── output
|   └── report.csv
├── insight_testsuite
    └── tests
        └── test_1
        |   ├── input
        |   │   └── complaints.csv
        |   |__ output
        |   │   └── report.csv
        ├── your-own-test_1
            ├── input
            │   └── complaints.csv
            |── output
                └── report.csv
                
python3.8 ./SRC/InsightCodingChallenge.py ./input/dataIn.csv ./output/dataOut.csv
