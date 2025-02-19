# conftest.py
import pytest
from decimal import Decimal
from faker import Faker # type: ignore
from calculator.operations import add, subtract, multiply, divide

fake_data = Faker()

def create_data_samples(sample_count):
    """Generate and yield random test data including values and operations."""
    operations_dict = {
        'add': add,
        'subtract': subtract,
        'multiply': multiply,
        'divide': divide
    }

    index = 0
    while index < sample_count:
        # Create two random values, one of which could be a single-digit for variability
        first_value = Decimal(fake_data.random_number(digits=3))
        second_value = Decimal(fake_data.random_number(digits=3)) if index % 5 != 0 else Decimal(fake_data.random_number(digits=1))
        
        # Select a random operation from the operations dictionary
        operation_key = fake_data.random_element(elements=list(operations_dict.keys()))
        operation_func = operations_dict[operation_key]

        # Prevent division by zero if the operation is division
        if operation_func == divide and second_value == Decimal('0'):
            second_value = Decimal('1')

        # Determine the expected result or error
        try:
            if operation_func == divide and second_value == Decimal('0'):
                result = "ZeroDivisionError"
            else:
                result = operation_func(first_value, second_value)
        except ZeroDivisionError:
            result = "ZeroDivisionError"

        yield first_value, second_value, operation_key, operation_func, result
        index += 1

def pytest_addoption(parser):
    """Add an option to specify the number of records to generate via command line."""
    parser.addoption("--num_records", action="store", default=5, type=int, help="Number of test cases to generate")

def pytest_generate_tests(metafunc):
    """Inject generated test data into the test functions."""
    if {"a", "b", "expected"}.intersection(set(metafunc.fixturenames)):
        num_records = metafunc.config.getoption("num_records")
        test_data = list(create_data_samples(num_records))

        # Adjust the structure of the parameters for flexibility with operation names
        data_for_parametrize = [
            (first_value, second_value, op_key if 'operation_name' in metafunc.fixturenames else op_func, result)
            for first_value, second_value, op_key, op_func, result in test_data
        ]

        metafunc.parametrize("a,b,operation,expected", data_for_parametrize)

