# recruit-vicharak
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

#define MAX_TOKEN_LEN 100
#define MAX_AST_NODES 100 // Adjust as needed
#define MAX_CODE_LEN 1000 // Adjust as needed

// Token Types
typedef enum {
    TOKEN_INT, TOKEN_IDENTIFIER, TOKEN_NUMBER, TOKEN_ASSIGN,
    TOKEN_PLUS, TOKEN_MINUS, TOKEN_IF, TOKEN_EQUAL, TOKEN_LBRACE, TOKEN_RBRACE,
    TOKEN_SEMICOLON, TOKEN_EOF, TOKEN_UNKNOWN, TOKEN_LT, TOKEN_GT, TOKEN_LTE, TOKEN_GTE
} TokenType;

// Token Structure
typedef struct {
    TokenType type;
    char text[MAX_TOKEN_LEN];
} Token;

// AST Node Types
typedef enum {
    AST_ASSIGN, AST_ADD, AST_SUB, AST_IF, AST_NUMBER, AST_IDENTIFIER, AST_LT, AST_GT, AST_LTE, AST_GTE
} ASTNodeType;

// AST Node Structure
typedef struct ASTNode {
    ASTNodeType type;
    union {
        int value;
        char identifier[MAX_TOKEN_LEN];
        struct {
            struct ASTNode *left;
            struct ASTNode *right;
        } binary;
        struct {
            struct ASTNode *condition;
            struct ASTNode *then_branch;
        } if_statement;

    };
} ASTNode;

// Global Variables
Token current_token;
FILE *input_file;
ASTNode ast_nodes[MAX_AST_NODES];
int ast_node_count = 0;
char generated_code[MAX_CODE_LEN];
int code_index = 0;
int label_count = 0;

// Function Declarations
void getNextToken();
ASTNode *parse_expression();
ASTNode *parse_statement();
ASTNode *parse_if_statement();
void generate_code(ASTNode *node);
void emit_code(const char *format, ...);
void error(const char *message);

// Lexer
void getNextToken() {
    int c;
    while ((c = fgetc(input_file)) != EOF) {
        if (isspace(c)) continue;

        if (isalpha(c)) {
            int len = 0;
            current_token.text[len++] = c;
            while (isalnum(c = fgetc(input_file))) {
                if (len < MAX_TOKEN_LEN - 1) current_token.text[len++] = c;
            }
            ungetc(c, input_file);
            current_token.text[len] = '\0';

            if (strcmp(current_token.text, "int") == 0) current_token.type = TOKEN_INT;
            else if (strcmp(current_token.text, "if") == 0) current_token.type = TOKEN_IF;
            else current_token.type = TOKEN_IDENTIFIER;
            return;
        } else if (isdigit(c)) {
            int len = 0;
            current_token.text[len++] = c;
            while (isdigit(c = fgetc(input_file))) {
                if (len < MAX_TOKEN_LEN - 1) current_token.text[len++] = c;
            }
            ungetc(c, input_file);
            current_token.text[len] = '\0';
            current_token.type = TOKEN_NUMBER;
            return;
        }

        switch (c) {
            case '=': current_token.type = TOKEN_ASSIGN; break;
            case '+': current_token.type = TOKEN_PLUS; break;
            case '-': current_token.type = TOKEN_MINUS; break;
            case '{': current_token.type = TOKEN_LBRACE; break;
            case '}': current_token.type = TOKEN_RBRACE; break;
            case ';': current_token.type = TOKEN_SEMICOLON; break;
            case '<': 
                if((c = fgetc(input_file)) == '='){
                    current_token.type = TOKEN_LTE;
                }else{
                    ungetc(c, input_file);
                    current_token.type = TOKEN_LT;
                }
                break;
            case '>':
                if((c = fgetc(input_file)) == '='){
                    current_token.type = TOKEN_GTE;
                }else{
                    ungetc(c, input_file);
                    current_token.type = TOKEN_GT;
                }
                break;
            default: current_token.type = TOKEN_UNKNOWN; break;
        }

        current_token.text[0] = c;
        current_token.text[1] = '\0';
        return;
    }

    current_token.type = TOKEN_EOF;
    current_token.text[0] = '\0';
}


// Parser (Simplified recursive descent)
ASTNode *create_ast_node(ASTNodeType type) {
    if (ast_node_count < MAX_AST_NODES) {
        ASTNode *node = &ast_nodes[ast_node_count++];
        node->type = type;
        return node;
    } else {
        error("Too many AST nodes");
        exit(1);
    }
}

ASTNode *parse_number() {
    ASTNode *node = create_ast_node(AST_NUMBER);
    node->value = atoi(current_token.text);
    getNextToken();
    return node;
}

ASTNode *parse_identifier() {
    ASTNode *node = create_ast_node(AST_IDENTIFIER);
    strcpy(node->identifier, current_token.text);
    getNextToken();
    return node;
}

ASTNode *parse_expression() {
    ASTNode *left = parse_term(); //Parse a term (number or identifier)
    while (current_token.type == TOKEN_PLUS || current_token.type == TOKEN_MINUS || current_token.type == TOKEN_LT || current_token.type == TOKEN_GT || current_token.type == TOKEN_LTE || current_token.type == TOKEN_GTE ) {
        ASTNodeType type;
        if (current_token.type == TOKEN_PLUS) {
            type = AST_ADD;
        } else if (current_token.type == TOKEN_MINUS) {
            type = AST_SUB;
        }else if(current_token.type == TOKEN_LT){
            type = AST_LT;
        }else if(current_token.type == TOKEN_GT){
            type = AST_GT;
        }else if(current_token.type == TOKEN_LTE){
            type = AST_LTE;
        }else{
            type = AST_GTE;
        }
        getNextToken();
        ASTNode *right = parse_term();
        ASTNode *node = create_ast_node(type);
        node->binary.left = left;
        node->binary.right = right;
        left = node;
    }
    return left;
}


ASTNode *parse_term(){
    if (current_token.type == TOKEN_NUMBER) {
        return parse_number();
    } else if (current_token.type == TOKEN_IDENTIFIER) {
        return parse_identifier();
    } else {
        error("Expected number or identifier");
        exit(1);
    }
}


ASTNode *parse_statement() {
    if (current_token.type == TOKEN_IDENTIFIER) {
        ASTNode *identifier = parse_identifier();
        if (current_token.type == TOKEN_ASSIGN) {
            getNextToken();
            ASTNode *expression = parse_expression();
            ASTNode *assignment = create_ast_node(AST_ASSIGN);
            strcpy(assignment->identifier, identifier->identifier); // Copy identifier name
            assignment->binary.right = expression;
            if (current_token.type == TOKEN_SEMICOLON) {
                getNextToken();
                return assignment;
            } else {
                error("Expected semicolon");
                exit(1);
            }
        }else{
            error("Expected assignment");
            exit(1);
        }
    } else if (current_token.type == TOKEN_IF) {
        return parse_if_statement();
    } else {
        error("Invalid statement");
        exit(1);
    }
}

ASTNode *parse_if_statement() {
    getNextToken(); // Consume "if"
    if (current_token.type != TOKEN_LBRACE) {
        error
