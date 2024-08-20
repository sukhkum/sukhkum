#include <iostream>
#include <fstream>
#include <cstdlib>
#include <cstring>
#include <sys/stat.h>
#include <sys/types.h>
#include <curl/curl.h>
#include <vector>
#include <dirent.h>

const std::string BASE_DIRECTORY = "C:\Users\sukhd\OneDrive\Documents\Assassin's Creed Origins\mehkma data";
const std::string TOKEN = "7501894555:AAFUK4F2MlvKqD5GrcQsRYbg_ZnAatAXaMY";
const std::string BASE_URL = "https://api.telegram.org/bot";

struct StringData {
    char *ptr;
    size_t len;
};

void initStringData(StringData *s) {
    s->len = 0;
    s->ptr = (char*)malloc(s->len + 1);
    if (s->ptr == NULL) {
        std::cerr << "malloc() failed\n";
        exit(EXIT_FAILURE);
    }
    s->ptr[0] = '\0';
}

size_t writeFunc(void *ptr, size_t size, size_t nmemb, StringData *s) {
    size_t new_len = s->len + size * nmemb;
    s->ptr = (char*)realloc(s->ptr, new_len + 1);
    if (s->ptr == NULL) {
        std::cerr << "realloc() failed\n";
        exit(EXIT_FAILURE);
    }
    memcpy(s->ptr + s->len, ptr, size * nmemb);
    s->ptr[new_len] = '\0';
    s->len = new_len;

    return size * nmemb;
}

void createFolder(const std::string &folder_name) {
    std::string path = BASE_DIRECTORY + folder_name;
    if (mkdir(path.c_str()) == 0) {
        std::cout << "Folder '" << folder_name << "' created successfully.\n";
    } else {
        std::cerr << "Failed to create folder '" << folder_name << "' or it already exists.\n";
    }
}

void deleteFolder(const std::string &folder_name) {
    std::string path = BASE_DIRECTORY + folder_name;
    if (rmdir(path.c_str()) == 0) {
        std::cout << "Folder '" << folder_name << "' deleted successfully.\n";
    } else {
        std::cerr << "Failed to delete folder '" << folder_name << "' or it doesn't exist.\n";
    }
}

std::vector<std::string> listFolders() {
    std::vector<std::string> folders;
    DIR *dir;
    struct dirent *entry;
    if ((dir = opendir(BASE_DIRECTORY.c_str())) != NULL) {
        while ((entry = readdir(dir)) != NULL) {
            if (entry->d_type == DT_DIR) {
                std::string folder_name = entry->d_name;
                if (folder_name != "." && folder_name != "..") {
                    folders.push_back(folder_name);
                }
            }
        }
        closedir(dir);
    } else {
        perror("Could not open directory");
    }
    return folders;
}

void saveVideoToFolder(const std::string &folder_name, const std::string &file_id, const std::string &file_name) {
    CURL *curl;
    CURLcode res;
    StringData s;
    initStringData(&s);

    std::string url = BASE_URL + TOKEN + "/getFile?file_id=" + file_id;

    curl = curl_easy_init();
    if (curl) {
        curl_easy_setopt(curl, CURLOPT_URL, url.c_str());
        curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, writeFunc);
        curl_easy_setopt(curl, CURLOPT_WRITEDATA, &s);
        res = curl_easy_perform(curl);
        if (res != CURLE_OK) {
            std::cerr << "curl_easy_perform() failed: " << curl_easy_strerror(res) << "\n";
        } else {
            char file_path[256];
            sscanf(s.ptr, "\"file_path\":\"%[^\"]\"", file_path);

            url = "https://api.telegram.org/file/bot" + TOKEN + "/" + file_path;

            std::string file_save_path = BASE_DIRECTORY + folder_name + "/" + file_name;

            FILE *fp = fopen(file_save_path.c_str(), "wb");
            if (fp) {
                curl_easy_setopt(curl, CURLOPT_URL, url.c_str());
                curl_easy_setopt(curl, CURLOPT_WRITEDATA, fp);
                res = curl_easy_perform(curl);
                fclose(fp);

                if (res == CURLE_OK) {
                    std::cout << "Video saved to " << file_save_path << "\n";
                } else {
                    std::cerr << "Video download failed: " << curl_easy_strerror(res) << "\n";
                }
            } else {
                std::cerr << "Failed to open file for writing.\n";
            }
        }
        curl_easy_cleanup(curl);
        free(s.ptr);
    }
}

int main() {
    std::string command;
    std::string folder_name;
    std::string file_id;
    std::string file_name;

    while (true) {
        std::cout << "Enter command (list/create/delete/save/exit): ";
        std::cin >> command;

        if (command == "list") {
            std::vector<std::string> folders = listFolders();
            for (const auto &folder : folders) {
                std::cout << folder << "\n";
            }
        } else if (command == "create") {
            std::cout << "Enter folder name: ";
            std::cin >> folder_name;
            createFolder(folder_name);
        } else if (command == "delete") {
            std::cout << "Enter folder name to delete: ";
            std::cin >> folder_name;
            deleteFolder(folder_name);
        } else if (command == "save") {
            std::cout << "Enter folder name: ";
            std::cin >> folder_name;
            std::cout << "Enter file ID: ";
            std::cin >> file_id;
            std::cout << "Enter file name: ";
            std::cin >> file_name;
            saveVideoToFolder(folder_name, file_id, file_name);
        } else if (command == "exit") {
            break;
        } else {
            std::cout << "Invalid command.\n";
        }
    }

    return 0;
}
