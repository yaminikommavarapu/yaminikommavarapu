package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"time"
)

type SortRequest struct {
	ToSort [][]interface{} json:"to_sort"
}

type SortResponse struct {
	SortedArrays [][]interface{} json:"sorted_arrays"
	TimeNs       int64             json:"time_ns"
}

func main() {
	http.HandleFunc("/process-single", processSingle)
	http.HandleFunc("/process-concurrent", processConcurrent)
	log.Fatal(http.ListenAndServe(":8000", nil))
}

func processSingle(w http.ResponseWriter, r *http.Request) {
	var req SortRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	start := time.Now()
	sortedArrays := make([][]interface{}, len(req.ToSort))
	for i, arr := range req.ToSort {
		sortedArrays[i] = sortArray(arr)
	}
	elapsed := time.Since(start)

	response := SortResponse{
		SortedArrays: sortedArrays,
		TimeNs:       elapsed.Nanoseconds(),
	}

	if err := json.NewEncoder(w).Encode(response); err != nil {
		log.Println("Error encoding response:", err)
		return
	}
}

func processConcurrent(w http.ResponseWriter, r *http.Request) {
	var req SortRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	start := time.Now()
	sortedArrays := make([][]interface{}, len(req.ToSort))
	ch := make(chan []interface{}, len(req.ToSort))

	for i, arr := range req.ToSort {
		go func(i int, arr []interface{}) {
			sortedArrays[i] = sortArray(arr)
			ch <- sortedArrays[i]
		}(i, arr)
	}

	for range req.ToSort {
		<-ch
	}

	elapsed := time.Since(start)

	response := SortResponse{
		SortedArrays: sortedArrays,
		TimeNs:       elapsed.Nanoseconds(),
	}

	if err := json.NewEncoder(w).Encode(response); err != nil {
		log.Println("Error encoding response:", err)
		return
	}
}

// Implement your chosen sorting algorithm here (e.g., bubble sort, merge sort)
func sortArray(arr []interface{}) []interface{} {
	// ... sorting logic ...
	return sortedArray
}
from flask import Flask, request, jsonify
from time import time
from concurrent.futures import ThreadPoolExecutor

app = Flask(_name_)

@app.route('/process-single', methods=['POST'])
def process_single():
    try:
        data = request.get_json()
        to_sort = data['to_sort']
    except KeyError:
        return jsonify({'error': 'Missing required fields'}), 400

    start_time = time()
    sorted_arrays = []
    for arr in to_sort:
        sorted_arrays.append(sorted(arr))
    execution_time = time() - start_time

    return jsonify({
        'sorted_arrays': sorted_arrays,
        'time_ns': int(execution_time * 1e9),
    })

@app.route('/process-concurrent', methods=['POST'])
def process_concurrent():
    try:
        data = request.get_json()
        to_sort = data['to_sort']
    except KeyError:
        return jsonify({'error': 'Missing required fields'}), 400

    start_time = time()
    with ThreadPoolExecutor(max_workers=None) as executor:
        sorted_arrays = list(executor.map(sorted, to_sort))
    execution_time = time() - start_time

    return jsonify({
        'sorted_arrays': sorted_arrays,
        'time_ns': int(execution_time * 1e9),
    })

if _name_ == '_main_':
    app.run(debug=True, port=5000)
