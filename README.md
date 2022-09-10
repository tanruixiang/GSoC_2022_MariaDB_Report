# Google Summer of Code 2022 Report
## Relevant Links
- **Organization:** [MariaDB Foundation](https://github.com/mariadb-corporation?type=source)
- **Mentor** [Rucha Deodhar](https://github.com/mariadb-RuchaDeodhar) and [Oleksandr Byelkin](https://github.com/sanja-byelkin)
- **Product:** [Pull Request:  MDEV-26182: Implement JSON_INTERSECT() ](https://github.com/MariaDB/server/pull/2261)
- **JIRA:** [MDEV-26182](https://jira.mariadb.org/browse/MDEV-26182)
## Introduction
This project will add the JSON_INTERSECT() function. This function can get the intersection between two jsons. We will try to implement this function in a way with low time and space complexity.

The behavior of intersection of different types is different. Taking the intersection of two objects is taking their common KV pairs. The intersection of array and array is to take their common elements. There is no intersection between scalar and object. An object and a scalar take an intersection with an array. As long as the object or scalar exists in the array, the intersection is the object or the scalar.

We put a json into the hash, scan another json, and efficiently get the intersection by reading and updating the hash information.
## Results Display
```
Example 1

SET @json1 = '{"key1":"value1","key2":[1,2,3],"key3":{"kkey3":"vvalue3"},"key4":5}';
SET @json2 = '{"key1":"value1","key2":[1,2,3],"key3":{"kkey3":"vvalue3"},"key4":3}';
SELECT JSON_INTERSECT(@json1, @json2);
+---------------------------------------------------------------------+
| JSON_INTERSECT(@json1, @json2)                                      |
+---------------------------------------------------------------------+
| {"key1": "value1", "key2": [1, 2, 3], "key3": {"kkey3": "vvalue3"}} |
+---------------------------------------------------------------------+
```
```
Example 2

SET @json1 = '[{"k2":"v2","k1":"v1"},{"k2":"v2","k1":"v1"},{"k2":"v2","k1":"v1"},1,"a","a"]';
SET @json2 = '[{"k1":"v1","k2":"v2"},{"k1":"v1","k2":"v2"},1.0,"a","a","a"]';
SELECT JSON_INTERSECT(@json1, @json2) as Intersection;
+---------------------------------------------------------------------+
| Intersection                                                        |
+---------------------------------------------------------------------+
| [{"k1": "v1", "k2": "v2"}, {"k1": "v1", "k2": "v2"}, 1.0, "a", "a"] |
+---------------------------------------------------------------------+
```
```
Example 3

SET @json1 = '{"key1":"value1"}';
SET @json2 = '{"key1":[1,2,3]}';
SELECT JSON_INTERSECT(@json1, @json2);
+--------------------------------+
| JSON_INTERSECT(@json1, @json2) |
+--------------------------------+
| NULL                           |
+--------------------------------+
```
## Implementation Details
First we introduce the parameter compare_whole.
- If compare_whole is true(When the array or object or scala is not at the outermost layer, compare_whole is true.), it means that the two jsons being compared, whether they are scalars, arrays or objects, must be exactly equal to have an intersection. Different types indicate that there is no intersection between jsons. If two scalars are compared, there is an intersection between them, which means that the two scalars are exactly the same. If two arrays are compared, there is an intersection between them, which means that the values at all indices of the two arrays are equal. If two objects are compared, there is an intersection between them, which means that all KV pairs of the two objects are equal.
- if compare_whole is false. When a value is a subset of another value, there is an intersection. For example, taking the intersection of two objects is taking their common KV pairs. The intersection of array and array is to take their common elements. An object and a scalar take an intersection with an array. As long as the object or scalar exists in the array, the intersection is the object or the scalar.

The specific implementation is as follows:
When comparing between json1 and json2, there can be following cases (When
there is an intersection, the intersection will be added to str) :
-  1 When at least one of the two json documents is of scalar type:
   -  1.1 If json1 and json2 both are scalar and they have same type and value, indicate that there is an intersection between them. The intersection is this scalar. Else, there is no intersection.
   -  1.2 If json1 is scalar but other is array (or vice versa):
       - 1.2.1  if compare_whole = true:
            Indicate that there is no intersection between them.
       - 1.2.2 if compare_whole = false:
            If the array has at least one element of same type and value as scalar,
            indicate that there is an intersection between them. The intersection
            is this scalar. Else there is no intersection between them.
   - 1.3 If one is scalar and other is object, indicate that there is no
          intersection between them, because they can not be compared.

-  2 When json1 and json2 are non-scalar type:
   - 2.1  If both jsons are arrays:
          Use hash to store all the normalized values in the json1 array
          and count the number of the same values.
          Scan the values in json2 and normalize them, find out whether there
          are equal values in the hash. Every time the same value is found in
          the hash table, the count in the hash table need to be updated.
      - 2.1.1 compare_whole = true:
            If the values of all indices in the two arrays are exactly equal,
            there is an intersection between them, and the intersection is any
            one of these two arrays. Else there is no intersection between them.
      - 2.1.2 compare_whole = false:
            If there are equal values at any index of two arrays, then there
            is an intersection between them. The intersection is the longest
            array that can be obtained by matching equal values between two
            arrays. Else there is no intersection between them.
   - 2.2  If both jsons are objects:
          Use hash to store all the key-value pairs in the json1 object.
          Scan the key-value pair in json2. Find out if there is the same key
          in the hash. If yes, take out the value in the corresponding key and
          normalize it. Then compare it with the value(normalization is also
          required) in the json2's key-value pair.
        - 2.2.1 compare_whole = true:
            If all KV pairs of two objects are equal, it means that there is an
            intersection between them, and the intersection is any one of these
            two objects. Else, there is no intersection between them.
        - 2.2.2 compare_whole = false:
            If there are equal KV pairs for two objects, it means that there is
            an intersection between them, and the intersection is all the equal
            KV pairs of these two objects. Else, there is no intersection
            between them.
   - 2.3  if one of the jsons is an object and the other is an array:
      - 2.3.1 compare_whole = true:
            Indicate that there is no intersection between them.
      - 2.3.2 compare_whole = false:
            Iterate over the array, if an element of type object is found,
            then compare it with the object. If the entire object matches
            i.e all they key value pairs match, indicates that there is an
            intersection between them, and the intersection is this object.
            Else, there is no intersection between them.
